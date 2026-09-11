# Kanban worker silent crash-loop (the operator lab, 2026-07-21)

## Symptom
A kanban card assigned to a local-model worker (e.g. `coder` = qwen3.8:27b-132k on M2)
gets dispatched by the gateway, the worker process spawns, runs for ~5 minutes
(~307s), then VANISHES with NO traceback, NO error in the worker log, and ZERO file
writes. The gateway's dispatcher auto-respawns it on its ~60s interval -> it dies again
-> 15+ consecutive `crashed` runs, all ~307s apart, all empty. The card sits `running`
forever, burning cycles, and you only notice because file timestamps never move.

## Evidence captured
- `kanban_show <id>` runs table: 15+ runs, every `outcome='crashed'`, `started_at` ->
  `ended_at` ~307s apart, `summary`/`error` empty, no files written to the workspace.
- Worker log (`~/.agent-home/profiles/<p>/logs/agent.log`, surfaced via `/api/logs/<p>`)
  shows NO exception, NO traceback, NO OOM message — the process just stops.
- M2 model (qwen3.8:27b-132k, 27.8B Q4_K_M, ~23GB VRAM, 196K ctx) confirmed healthy via
  `/api/ps` — not a model-load failure.

## Root cause (most likely)
Process-level silent kill on long-running, memory-/context-heavy agent loops on a local
model: OOM (the 27B Q4 weighs ~23GB; a worker stacking system prompt + tool outputs +
long reasoning can exceed the resident budget and get SIGKILLed by the kernel with no
message) or context-window overflow inside the agent loop. Either way it presents as a
clean silent PID death at ~5min, NOT a Python exception. Config timeouts were RULED OUT:
- `max_runtime_seconds` was NULL on the task -> `enforce_max_runtime` (needs non-null) never fires.
- `dialog_timeout_s` is defined in config.yaml but never READ by the dispatcher/worker.
- Claim TTL is 900s -> the ~307s death is far below reclaim, so it isn't a stale-claim reclaim.
- No exceptions in the log -> not a Python error in the task code.

## The fix that worked
DO NOT try to make the worker finish the big task. For multi-file / >5min builds the
LEAD (the main session on M1) implements directly. Workers are fine for SMALL tasks that
finish well under 5 min (S1/S2-class work completed fine that way).

Concretely:
1. `hermes kanban block <id>` so the gateway dispatcher SKIPS the card on the next
   interval — this halts the crash loop (the dispatcher skips `blocked` cards).
2. Verify no live worker proc: `ps -ef | grep -E "hermes -p <profile>" | grep -v bash`
   should show nothing after ~1 min. (A defunct `[hermes] <defunct>` is a harmless
   already-dead child awaiting gateway reaping — not a respawn.)
3. Do the implementation YOURSELF with direct tool calls (read_file / patch /
   write_file), keeping the kanban card as a tracking artifact — update its summary via
   `kanban_comment` (or `kanban_complete` once truly done).
4. If multiple cards touch the SAME file (e.g. several MC stages all edit
   `static/js/app.js`), block all but the one you're actively writing, then release them
   sequentially after each stage is code-verified — avoids last-writer-wins clobbering
   (see hermes-kanban-fleet Pitfall 12).

## When to suspect this BEFORE dispatching
- Card body implies: edit 3+ files, or a feature needing sustained reasoning, or a build
  that previously took you >5 min. -> Implement directly; don't `dispatch`.
- Card is `running` with `consecutive_failures` climbing and no file output after a few
  minutes -> it's looping; block it.

## Why "dispatch a subagent to look into it" ALSO fails
A `delegate_task` subagent runs the SAME local model and hits the SAME ~5min silent death
when the investigation needs long sustained work. So "dispatch someone to investigate the
runtime" is not viable here — do the runtime investigation as lead too (or scope it to a
<5min cron-style probe). This is a hard environment constraint, not a one-off.
