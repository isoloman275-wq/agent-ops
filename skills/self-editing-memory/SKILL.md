---
name: self-editing-memory
description: "Use when memory is full. Letta-style self-edit blocks."
version: 1.0.0
author: the lab
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [memory, letta, memgpt, self-editing, curation]
    category: memory
---

# Self-Editing Memory Blocks (Letta pattern)

The agent curates its OWN memory mid-session via block operations instead of
hitting the 99,999-char wall and doing manual surgery afterwards.

## Tool

`~/.agent-home/scripts/memory_block.py` — manages `~/.agent-home/memories/MEMORY.md`
(§-separated entries = blocks, budget 99,999 chars).

```bash
M=~/.agent-home/scripts/memory_block.py
python3 $M stats                # total/free chars + largest blocks
python3 $M list | head -20      # id, size, preview of every block
python3 $M find "QUERY"         # locate a block before editing
python3 $M append "TEXT"        # add block (REFUSED exit 2 if over budget)
python3 $M replace ID "TEXT"    # rewrite block by id (budget-checked)
python3 $M remove ID            # delete block by id
python3 $M trim 3 --dry         # preview removing 3 largest
python3 $M trim 3               # remove 3 largest (refile survivors to fact_store)
```

Exit 2 = budget/usage error → run `stats`, then `trim` or consolidate.
Writes are atomic (tmp+rename) with a `.bak` kept each write.

## The workflow (when memory is near-full)

1. `stats` → how much over/free.
2. `list` → identify STALE blocks (old dates, superseded state, done projects).
3. Before removing anything load-bearing: `fact_store add` the durable core
   (procedures/lessons live in skills, facts in fact_store — MEMORY.md is only
   the hot layer).
4. `remove`/`replace` the stale blocks; `append` the one new fact.
5. `stats` to confirm the result.

## HARD RULES

- NEVER use the `memory` tool and this script on the same file in the same
  turn without re-reading `stats` between — IDs shift after every write.
- Never delete a operator correction / hard rule / GPU-NODE-safety block without its
  content being preserved in fact_store or a skill first.
- One write per decision — re-run `find` before each replace/remove.

## Provenance

Pattern borrowed from Letta/MemGPT: core memory blocks that the agent edits
via `core_memory_append` / `core_memory_replace`, plus archival (our fact_store)
and recall (session_search). The one gap in Hermes was the self-edit tool; this
is it. Built+verified 2026-09-11 (caught the literal-§-in-text parsing trap:
split ONLY on separator lines `(?m)^§$`, never the bare character).
