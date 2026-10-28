> **ARCHIVED COPY** — the `evidence-before-claim` skill was absorbed into the
> `ultron-execution-discipline` umbrella (2026-08-23 curation pass), and Rules 1–4
> are now **enforced at runtime** by the `verify-before-claim` engine.
> Paths below are rewritten: bare filenames are siblings in this same
> `references/` directory; `../scripts|templates/...` live one level up in the umbrella.

name: evidence-before-claim
description: Verify-before-claim + no-relaunch discipline for the operator.
version: 2026-08-24
author: the lab (the operator / Hermes)

# Evidence Before Claim

the operator has zero tolerance for unverified assertions. This is the single most-repeated
correction in the lab's history. Encode it as reflex, not intention.

## The rule

### 1–4. Now Automated → See `verify-before-claim` engine

Rules 1–4 are enforced at runtime by [`verify_guard.py`](../../verify-before-claim/verify_guard.py):

- **Proof precedes the claim** → engine runs live probes before every response
- **Empty probe ≠ nonexistence** → engine never concludes "doesn't exist" from one probe
- **"Is X safe?" → verify live** → engine always checks process + GPU + models together
- **After correction → re-verify first** → engine auto-reprobes, not memory

Full engine: `skills/devops/verify-before-claim/verify_guard.py`

### 5. HALT ON USER VETO (human-only — cannot be automated safely)

When the user vetoes an action:
- **NO** / **stop** / **don't** / **"I don't want that"** / **"not going there"**

You MUST:
1. **Kill it immediately** if running — do not let it finish gracefully
2. **Do NOT re-launch** it, do NOT continue it, do NOT ask to proceed
3. **Stand down completely** until they explicitly say go

> Re-launching after a veto is the single worst offense in this lab.
> the operator has corrected this repeatedly: "launched MAIN-NODE train after explicit
> 'no', then again — multiple times, each caught"

> NOTE: If the user says a process/layer is "dead weight" or "unused",
> that is NOT authorization to kill it — he means USE IT. Only kill a
> process he explicitly says to remove. When in doubt, restart/keep
> and wire it in instead of pruning.

### Why This Needs a Human Rule

An automated guard can't distinguish "the user wants me to stop THIS
specific action" from "the user wants me to stop all related work."
The context of a veto — whether it's a hard stop, a redirect, or a
conservative pause — requires human interpretation.

## Anti-patterns (never)

- Stating a verdict from a single probe with a too-short timeout
- Confessing to a fault you cannot confirm ("I killed it") without evidence
- Re-asserting a claim after "you didn't verify it" instead of running the check
- Using "no"/"stop" corrections as an excuse to abandon the task entirely

## Verification pattern (reuse for any "what is the state of X")

- process: `ps -eo pid,etimes,args | grep -iE "<pattern>" | grep -v grep`
- GPU: `nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv,noheader`
- resident models: `curl -s --max-time 15 http://<host>:11434/api/ps`
- Run all three together before declaring anything dead/alive.