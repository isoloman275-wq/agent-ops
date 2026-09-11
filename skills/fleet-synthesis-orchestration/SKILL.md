---
name: fleet-synthesis-orchestration
description: Merge parallel delegate_task outputs into one deliverable.
---

# Fleet Synthesis Orchestration

When a task is wide-scope and you decompose it across parallel subagents (e.g. 4 analyst lenses + 1 synthesizer), the default `delegate_task` fan-out only coordinates **timing**, not **data flow**. A synthesis child does NOT automatically receive its siblings' outputs. This is the #1 failure mode: the synthesizer runs first-principles and either fabricates or honestly flags "reports not injected" — both waste the fleet.

## The correct pattern (use EVERY time)

**Option A — chain via context_from (preferred for clean merges):**
Spawn leaf analysts first (batch). Then spawn the synthesizer as a SEPARATE `delegate_task` call with `context_from: [leaf_job_ids]`. The scheduler injects each leaf's most-recent completed output into the synthesizer's prompt. This is the only mechanism that guarantees the merge uses real sibling data.

**Option B — paste sibling outputs into the synthesis context:**
If you already have the leaf outputs in-thread (they returned to you), paste the relevant JSON/sections verbatim into the synthesizer child's `context` field. Do NOT summarise or paraphrase — paste the actual returned content so the child isn't re-inventing it.

**NEVER** dispatch a "synthesis" child in the same batch as its dependencies and assume it sees them. Same-batch children run concurrently; the synthesizer will finish before leaves if not gated.

## Mid-run correction discipline

When the user corrects a key input AFTER dispatch (e.g. "$135/week not /month"):
1. `delegate_task(action='steer')` ALL affected children — but know steering is **queued, not guaranteed**: if a child already completed, the steer is reported as `missed_steer` and silently lost.
2. Therefore: ALWAYS re-run the dependent synthesis child after a correction, even if you think the steer landed. A missed steer on one leaf poisons the merged output.
3. In the re-run synthesis context, explicitly state "use the CORRECTED figures; prior outputs may be stale on X" so the child doesn't average bad + good data.

## Honesty rule for synthesis children
If a synthesis child cannot access sibling data, it MUST say so explicitly and either (a) request re-injection, or (b) perform first-principles synthesis while flagging provenance — NEVER present estimated numbers as if they were the analysts' verified outputs. The parent (you) must catch this and re-run, not deliver the half-merged result as final.

## Verification before delivering a fleet result
- Confirm the synthesis output references specifics from EACH leaf (not generic restatement).
- If a leaf was steered mid-run, confirm the synthesis used corrected values (search the output for the corrected figure).
- If any child reported `missed_steer`, treat the consolidated deliverable as INVALID until re-run.

## Sequencing nuance (from the Aug 2026 life-plan session)
The full pipeline was THREE waves, not one fan-out: (1) analysis fan-out (4 lenses +
synthesis), (2) corrected re-run of synthesis after user input fixes, (3) a SEPARATE
live-data fan-out (search-based, different agents) to replace ESTIMATE tags. Keep these
as distinct dispatch batches. Do not try to fold live-data into the analysis batch — the
analysis agents should run on stated baselines; live verification is a follow-on wave so a
property-site timeout in wave 3 doesn't block the decision tree in wave 1.

## Pitfalls
- "They wait on each other" (control_hint) = timing only. Do not confuse with data sharing.
- Background batch completion note says "waited on each other" — that means they didn't START until deps finished IF chained; in a same-batch fan-out they do NOT share data.
- A child finishing in <60s will miss a steer queued after dispatch. Re-run the merge.
- User corrections in fleet contexts are often terse factual redirects ("$135 per week", "not the mother, my mother"). Act instantly via steer + re-run; do not over-acknowledge.
- A TIMEOUT (status=timeout, no summary) on a leaf is the same class of failure as missed_steer: the consolidated deliverable is INVALID until that leaf is re-run. In this session 2 of 3 live-data leaves timed out at 600s on full-site scrapes; the fix was a narrower search-based re-run (max 8 searches, 300s budget) which returned. Treat any leaf with no summary as a hard blocker, not a soft gap — re-dispatch with tighter scope before delivering.
