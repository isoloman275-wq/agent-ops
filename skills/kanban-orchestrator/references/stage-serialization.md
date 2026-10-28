# Stage Serialization — exact command sequence

Companion to the "## Staged multi-phase builds" section in SKILL.md. When several
cards' workers all edit the SAME file (e.g. multiple frontend/backend stages
touching one `app.js` / `server.py`), run them ONE AT A TIME. Parallel writes
clobber each other (last-writer-wins) and silently lose a whole stage.

## Step 0 — map the file-touch graph
Read each card body; note which files it touches. Any two cards sharing a file
must not run concurrently. (Also resolves GPU-NODE single-VRAM contention when two
cards share a model host.)

## Step 1 — defer every card except the first (reclaim → block → kill)
Do this for each card you want to hold back. Order matters:
`reclaim` only works while the card is `running`, so do it FIRST; `block` is the
durable dispatcher guard (dispatcher skips `blocked`), so do it before you kill
the worker to prevent a respawn during the kill.

```bash
# 1) release the claim while still running (cleans up the lock):
~/.local/bin/hermes kanban reclaim <deferred_id>
# 2) block so the dispatcher skips it (do this BEFORE killing the worker):
~/.local/bin/hermes kanban block <deferred_id> "staging: <prev> runs first; unblock after verified"
# 3) kill the already-spawned worker PID:
ps -ef | grep -E "hermes -p (coder|builder|architect|reviewer|...)" | grep -v grep | grep "<deferred_id>"
kill -9 <worker_pid>
```
If you blocked first by mistake, `reclaim` will report "not running" — that's
harmless; the block already prevents re-dispatch. Just proceed to kill.

Verify only the first stage is active:
```bash
python3 -c "import sqlite3;c=sqlite3.connect(os.path.expanduser('~/.agent-home/kanban.db'));[print(r) for r in c.execute(\"SELECT id,status,assignee FROM tasks WHERE id IN ('<s1>','<s2>','<s3>')\")]"
# expect: <s1>=running, <s2>/<s3>=blocked, and no worker procs for s2/s3
```

## Step 2 — verify stage N landed, THEN unblock stage N+1
Confirm true completion (don't trust a reaped background watcher's -15 exit):
```bash
python3 -c "import sqlite3,os;print(sqlite3.connect(os.path.expanduser('~/.agent-home/kanban.db')).execute(\"SELECT status FROM tasks WHERE id='<s1>'\").fetchone()[0])"
# did it actually write? grep a feature marker + check mtime:
grep -niE "FEATURE_MARKER" /path/to/file.js
stat -c '%y %n' /path/to/file.js
```
Only if the code landed: `~/.local/bin/hermes kanban unblock <next_id>` — the live
gateway dispatcher picks it up on the next ~60s tick. Repeat until all stages done,
then integrate + restart the service.

## Notes
- Background poll loops are SIGTERM'd after ~10 min (exit -15), so they die before
  the task finishes. Use them only as a heartbeat; detect completion via the DB query above.
- For a purely manual build (no gateway), drive each card with
  `hermes -p <profile> --skills kanban-worker chat -q "work kanban task <id>"`
  one at a time instead of unblocking.
