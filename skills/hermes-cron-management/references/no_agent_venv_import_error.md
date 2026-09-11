# no_agent `.py` cron: ModuleNotFoundError (wrong Python / venv mismatch) — Case C

## Symptom
Run log `~/.agent-home/cron/output/<job_id>/<ts>.md` shows:
```
Script exited with code 1
stderr:
Traceback (most recent call last):
  File "~/.agent-home/scripts/<name>.py", line N, in <module>
    import feedparser   # or pandas / yfinance / ccxt
ModuleNotFoundError: No module named 'feedparser'
```
The script IS present (Case A `Script not found` does NOT apply). It dies on import
because Hermes runs `no_agent` `.py` with **system python3**, which lacks the
project venv dependencies.

## Confirm
```
python3 -c "import feedparser"                                  # system — fails
/path/to/projects -c "import feedparser"   # venv — ok
```

## Fix — venv-wrapper `.sh`
`~/.agent-home/scripts/<name>.sh`:
```bash
#!/bin/bash
# Run <name>.py with the project venv (has the deps system python3 lacks).
cd /path/to/projects   # if .py uses repo-relative imports
exec /path/to/projects <name>.py
```
- `chmod +x ~/.agent-home/scripts/<name>.sh`
- Repoint cron `script` `<name>.py` → `<name>.sh` via **direct jobs.json edit**
  (the `cronjob` tool rejects a single-field `script` change: "No updates provided.").
  Match on `"id"`, set `"script": "<name>.sh"`, back up first. Scheduler reloads each tick.
- Verify at the SCHEDULER: `cronjob action=run job_id=<id>` → read newest
  `~/.agent-home/cron/output/<job_id>/` file: clean `Mode: no_agent (script)` + stdout,
  no `ModuleNotFoundError`.

## Real repair (2026-08-21, Project4 trading crons)
Three crons were red:
- `IFT Pullback Entry Watcher` (a12b029b8a09) — `Script not found`
  (the `.py` only lived in `project4-trading/execution/`, not `~/.agent-home/scripts/`).
- `US Pullback Entry Watcher` (c310e28505a5) — same `Script not found`.
- `Project4 Daily Briefing Aggregator` (94a74741e51b) — `ModuleNotFoundError: feedparser`.

Fix: created `ift_pullback_watch.sh`, `us_pullback_watch.sh`, `project4_aggregator.sh`
wrappers that `exec`
`/path/to/projects`, repointed the 3
crons via jobs.json edit. Verified: IFT/US ran (US even fired a real NVDA pullback
alert to Telegram), aggregator wrote its briefing, Telegram token valid (bot Hux275Bot).

## Prevention — health watchdog
Notification/monitor crons fail SILENTLY (error or paused) and the user notices only
the absence of alerts. Add a cron that checks the others' `last_status`/paused state
and pings Telegram on failure. See `scripts/event_watchdog.py` for the event-based
pattern (silent ticks, alert-only-on-event delivery).
