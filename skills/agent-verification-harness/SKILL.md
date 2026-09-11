---
name: agent-verification-harness
description: Use when improving agent reliability; wire verifier swarms.
---

# Agent Verification Harness (independent verifier > self-critique)

## The measured result (lab bench, 2026-08-24, deterministic oracle)

A/B/C benchmark on 10 verifiable tasks, graded by a Python oracle (no LLM grading bias):

| Condition | Accuracy | Latency | Verdict |
|---|---|---|---|
| A: bare cheap model (qwen3.5:2b-aux @ M3) | 9-10/10 | ~190s | baseline |
| B: worker + self-critique ("check your own answer") | 9-10/10 | ~2x | **zero gain — dead pattern** |
| C: worker + INDEPENDENT verifier (ornith:9b @ M1) | 9/10 | ~2x | **caught the blind-spot error every run** |

Key finding (reproduced 3 runs): on t05 (list primes < 20), the worker dropped "19".
Self-critique REPEATED its own error both times it occurred. The independent
verifier (different model, different machine) corrected it EVERY time it ran.
This reproduces the published "observer independence is decisive" finding on our
own hardware. Full data: `hermes-workspace/verify-bench/{verify_bench.py,RESULTS.md}`.

RULE OF THUMB: cheap model + independent verifier ≥ expensive model accuracy,
at commodity cost — but only with a healthy harness underneath. Refinement from
the 12-task capability ladder: verification reliably rescues CLASSIFICATION and
EXTRACTION misses — reasoning and long-context misses survive a second model's
check (measured 1/5 rescue overall). Wire verifiers on extraction/classify lanes;
on reasoning lanes spend the tokens on a better worker instead.

## Bench recipe (rerunnable)

1. Write N tasks with DETERMINISTIC oracles (`lambda answer: "391" in answer`), never LLM grading.
2. Condition A: single generate call. Condition B: generate with "think step by step, output FINAL:" then grade the final line. Condition C: generate as in B, then send task+proposed-answer to a DIFFERENT model on a DIFFERENT machine ("strict verifier; reply FINAL: <corrected answer>").
3. Grade all three conditions against the same oracles. Report pass-rate AND wall-time.
Script: `hermes-workspace/verify-bench/verify_bench.py` (hosts/model names at top).

## Wiring into Hermes kanban (done 2026-08-24)

`hermes kanban swarm --worker 'profile:title' --verifier independent-verifier --synthesizer <profile> '<goal>'`
- `~/.agent-home/profiles/independent-verifier/` = ornith:9b-64k @ ollama-m1, with an AGENT.md
  adversarial protocol: verify acceptance criteria line-by-line WITH TOOLS; claims without
  tool evidence are UNVERIFIED=FAIL; PASS only via kanban_complete metadata `{"gate":"pass"}`;
  FAIL via block with a numbered defect list. Judge, don't improve.
- The swarm's auto-generated verifier card gates completion — good enough; the AGENT.md adds
  the adversarial posture.

### CRITICAL PITFALL — provider fallback silently violates machine bans (hit first wiring test)
Worker profiles carry fallback providers (builder: primary ollama-m1, FALLBACK ollama-m2).
When primary is slow/wedged, the runtime silently routed to qwen3.8:27b ON M2 → OOM crash-loop
×12 and unauthorized load on a machine the user had banned that day. Checks before ANY dispatch:
1. Grep assignee profile config.yaml fallback chains vs current user machine-access rules;
   strip banned hosts (with user approval — profiles are his).
2. Pin hard cases per-task: `hermes kanban set-model <task_id> <model> <provider>`.
3. After spawn, READ THE FIRST LOG LINES for the resolved base_url — config alone lies.
4. Emergency stop when it hits a banned host: reclaim + archive ALL swarm cards (worker,
   verifier, synthesizer — archive fails while running, so reclaim first), kill orphaned PIDs,
   verify zero worker processes remain. M2 was unharmed (requests rejected, never allocated).

## Swarm-loop ownership (user rule for multi-node watch/swarm ops)

When a task needs a continuous watch/decide/act loop across nodes, the SWARM runs
the loop — the main agent does NOT hand-probe each step. Pattern (proven on a live
3-node mesh):
1. DRIVER NODE (LLM-free): ONE batched raw-state fetch per cycle (a single ssh/cmd
   probe returning all signals). Never per-signal probes — rate limiters and
   fail2ban punish retry spam, and sequential hand-probing is the failure this
   rule exists for.
2. WORKER TIER (multi-slot worker, parallel): extract meaning from raw signals,
   1 request per signal across slots.
3. ROUTER TIER (small model): classify phase -> WAIT / IDLE / PICKUP / EXECUTE / DEAD.
4. VERIFIER TIER (independent model+machine, parallel x2): AGREE/CORRECT gate on
   the classification before anything fires.
5. MAIN AGENT: reads the verdict file; ONLY it fires irreversible actions, and
   only on verified verdicts.
Corollary: if you notice you have hand-run every probe sequentially this session,
stop and build the driver loop instead of continuing manually.

## Per-backend request settings (thinking-off + endpoint preflight)

Qwen3-family models silently burn the whole token budget on reasoning unless
disabled PER REQUEST; a "failing" model is often a parsing bug, not a capability
gap. Match the field to the backend:
- llama-server :8080 (/v1): `chat_template_kwargs: {"enable_thinking": false}` and
  parse `message.content || message.reasoning_content` (thinking-only replies
  return empty content).
- Ollama native /api/generate: `think: false` at TOP LEVEL of the body — nesting
  it inside `options{}` is silently ignored and thinking leaks into the answer.
- OpenAI-compatible cloud: use the provider's disable field per model; on a 400,
  retry once WITHOUT provider-specific extras (some reject unknown params).
Before firing any batch lane: ONE warm-up request against the exact endpoint URL
the lane will use — remote callers must use the LAN IP, never localhost (a
localhost default produced a false 0/12 lane) — and confirm content is non-empty
with thinking off.

## Verifier Sizing Rule (NON-NEGOTIABLE)
**Agent-grade models MUST use `num_ctx` 65536 (64K minimum) — no more, no less.**
- Sub-64K variants (e.g. `ornith:9b-verifier` @ 8K) are SPECIAL-PURPOSE ONLY
  (tiny prompts, raw API callers) and can NEVER be a Hermes profile model
- Create via `/api/create` with a `parameters` dict (NOT a raw modelfile string):
  `{\"model\":\"ornith:9b-64k\",\"from\":\"ornith:9b\",\"parameters\":{\"num_ctx\":65536,\"num_gpu\":34}}`
- M3 ships num_ctx 64000 already — do not re-configure

## Related wedge lesson (M1 Windows Ollama reload-loop)

Verifier timeouts during the bench were NOT VRAM starvation: timed-out jobs sat in Ollama's
request queue and re-requested the model after each llama-server kill → reload-loop at 128K ctx
on an 8GB card. Fix: kill serve+llama-server processes (PowerShell Stop-Process), relaunch serve
(wipes queue), load the capped variant FIRST. Post-fix: 44.5 tok/s, ~2.3s/verification.

## What NOT to do

- Do not use self-critique as a quality mechanism — measured dead weight.
- Do not trust profile configs for machine routing — verify the resolved endpoint per run.
- Do not put a sub-64K-context model on any Hermes profile.
- Do not run the bench with LLM grading — oracle must be deterministic or the comparison is noise.
