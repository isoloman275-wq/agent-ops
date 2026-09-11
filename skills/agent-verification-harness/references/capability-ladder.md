# Capability Ladder — measured per-model limits (deterministic-oracle bench)

12-task ladder per model: 2 trivial, 4 reasoning, 3 structured-JSON, 3 needles
(2K/16K/60K tokens), all oracle-graded — never LLM-graded. Rerun from
`hermes-workspace/swarm-ladder/` (`swarm_ladder.py <tier>`, `aggregate.py` ->
REPORT.md). Numbers are from the latest full sweep — re-verify after ANY model
roster change; treat stale numbers as hypotheses.

## Ladder results

| Tier | Model | Pass | Median | 60K needle | Real misses |
|---|---|---|---|---|---|
| M3 | qwen3.5:2b-aux @64K | 11/12 | 13s | 375s pass | numeric sort-order |
| M3 | granite4.1:3b (deleted) | 8/12 | 4s | timeout | reversal, decimals, classification |
| M1 | ornith:9b @64K | 11/12 | 7s | ~350s marginal | needle timeout at 420s cap |
| M1 | ornith1.5:9b (deleted) | 11/12 | 4s | 354s pass | sort-order |
| M2-W | Qwen3.6-14B-A3B (worker) | 9/12 | 0.35s | 38s pass | trivia recall, reversal, sort |
| M2-B | Huihui-Qwen3.8-27B @196K | 11/12 | 1.35s | 78s pass | top-k selection |
| Cloud | glm-5.3-flash (zai) | 12/12 | 2.1s | pass | — |

M1 60K = marginal, not impossible: the ingest wall (~6ms/token, ~334s for 56K
tokens) puts 9B models near any sane timeout. Never route >16K-context prompts
to M1 when reliability matters.

## Concurrency (measured)

- M2 worker :8081 (4 slots): 8 simultaneous requests 8/8 correct, 1.2s wall.
- M1 ollama: 4 same-model parallel 4/4, 2.8s wall. A concurrent SECOND model
  load while one is resident = instant 500 storm — one model at a time on M1.
- M2 brain :8080: single slot (parallel 1) — serialize all brain work.
- Sustained swarm shape: <=4 tasks per node, ~8-10 workers lab-wide.

## Routing decision table (accuracy-first)

| Work type | Route to | Why |
|---|---|---|
| Bulk long-context extraction/drafts | M2 worker (14B) | fastest local, all needles pass; verify output downstream |
| Verification of classification/extraction | M1 ornith:9b | bench-proven verifier role on these lanes |
| Reasoning, JSON, code review | M1 ornith:9b | JSON 3/3, reasoning 4/4, 4-way parallel |
| Trivial/routing/booleans | M3 2b-aux | ~1.3s classify, cheapest lane |
| >16K-context prompts | M2 brain or cloud, NEVER M1 | ingest wall above |
| Long-context synthesis | M2 brain (explicit go only) | only fast local 60K lane |
| Irreducible hard problems | cloud glm-5.3-flash | 12/12 reference; cost-scoped to lead + children |

## Cloud rule

glm-5.3-flash via z.ai is the ONLY permitted cloud model (user rule — no other
provider or model until the user says). Endpoint is the billing switch:
`.../api/coding/paas/v4` draws plan quota, `.../api/paas/v4` draws balance — pin
cloud profiles to the coding endpoint. The general endpoint cannot disable
thinking (error 1210); the coding endpoint accepts it.
