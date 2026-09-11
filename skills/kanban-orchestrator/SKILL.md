---
name: kanban-orchestrator
description: Decomposition playbook + anti-temptation rules for an orchestrator profile routing work through Kanban. The "don't do the work yourself" rule and the basic lifecycle are auto-injected into every kanban worker's system prompt; this skill is the deeper playbook when you're specifically playing the orchestrator role.
version: 3.0.0
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [kanban, multi-agent, orchestration, routing]
    related_skills: [kanban-worker]
---

# Kanban Orchestrator — Decomposition Playbook

> The **core worker lifecycle** (including the `kanban_create` fan-out pattern and the "decompose, don't execute" rule) is auto-injected into every kanban process via the `KANBAN_GUIDANCE` system-prompt block. This skill is the deeper playbook when you're an orchestrator profile whose whole job is routing.

## Profiles are user-configured — not a fixed roster

Hermes setups vary widely. Some users run a single profile that does everything; some run a small fleet (`docker-worker`, `cron-worker`); some run a curated specialist team they've named themselves. There is **no default specialist roster** — the orchestrator skill does not know what profiles exist on this machine.

Before fanning out, you must ground the decomposition in the profiles that actually exist. The dispatcher silently fails to spawn unknown assignee names — it doesn't autocorrect, doesn't suggest, doesn't fall back. So a card assigned to `researcher` on a setup that only has `docker-worker` just sits in `ready` forever.

**Step 0: discover available profiles before planning.**

Use one of these:

- `hermes profile list` — prints the table of profiles configured on this machine. Run it through your terminal tool if you have one; otherwise ask the user.
- `kanban_list(assignee="<some-name>")` — sanity-check a single name. Returns an empty list (rather than an error) for an unknown assignee, so this only confirms a name you're already considering.
- **Just ask the user.** "What profiles do you have set up?" is a fine first turn when the goal needs more than one specialist.

Cache the result in your working memory for the rest of the conversation. Re-asking every turn wastes a tool call.

## When to use the board (vs. just doing the work)

Create Kanban tasks when any of these are true:

1. **Multiple specialists are needed.** Research + analysis + writing is three profiles.
2. **The work should survive a crash or restart.** Long-running, recurring, or important.
3. **The user might want to interject.** Human-in-the-loop at any step.
4. **Multiple subtasks can run in parallel.** Fan-out for speed.
5. **Review / iteration is expected.** A reviewer profile loops on drafter output.
6. **The audit trail matters.** Board rows persist in SQLite forever.

If *none* of those apply — it's a small one-shot reasoning task — use `delegate_task` instead or answer the user directly.

## Setup — standing up the fleet (do this BEFORE decomposing)
A Kanban board with only one profile can't fan out — every card lands on that one profile and runs serially. Multi-agent parallelism requires **>=2 named worker profiles** plus one orchestrator. Setup sequence (exact verified commands + the `providers.X` string trap are in `references/kanban-fleet-setup.md`):

1. **Discover what exists.** `hermes profile list` — if only `default` shows, you have nothing to route to yet.
2. **Decide the roster WITH the user.** Profile names are their call (don't invent them). A proven 4-role shape for a multi-stream dev shop:
   - `architect` (orchestrator/dev-lead) — decomposes, scales, finds new revenue avenues
   - `coder` — implementation
   - `researcher` — web/discovery/synthesis
   - `reviewer` — QA before handoff
3. **Create profiles** (model choice is the key decision — see VRAM rule below):
   `hermes profile create <name> --clone --description "..."`
   NOTE: `hermes profile create` has NO `--model` flag. Create with `--clone` (copies config/.env/skills) + a role-specific `--description` (the decomposer routes on the description), THEN set model+provider per-profile with `hermes -p <name> config set model/provider`. Also: `hermes config set providers.<x> '{...}'` stores the provider as a broken JSON *string* — you must re-parse it to a dict afterward (see `references/kanban-fleet-setup.md` for the exact, verified commands incl. this trap and the `ollama-m2` provider add).
4. **Wire the config** in `~/.agent-home/config.yaml` under `kanban:`:
   - `orchestrator_profile: architect` — auto-decompose routes through this profile
   - `default_assignee: coder` — fallback for unassigned cards
   - `dispatch_in_gateway: true` + `dispatch_interval_seconds: 60` (dispatcher already runs inside the gateway by default)
   - `auto_decompose: true` (the orchestrator breaks prompts into cards automatically)
5. **Verify the board + dispatcher.** `hermes kanban boards` (expect a `default` board), `hermes kanban diag` (expect "No active diagnostics"), confirm `kanban.db` exists at `~/.agent-home/kanban.db`.
6. **Live fan-out test.** Create 2 cards assigned to DIFFERENT profiles with no parent links; confirm the dispatcher (60s pass) claims + runs them, and watch via `hermes kanban tail <id>` or the dashboard.

### VRAM contention rule (critical for local-model fleets)
If two profiles point at **different models on the same Ollama host**, they CANNOT run simultaneously — they contend for one VRAM pool and one serializes. Implications:
- On a single-inference-box setup (e.g. M2 with `qwen3.8:27b-132k` + `glm-4.7-flash` both loaded), only ONE M2-profile runs at a time. To get true parallelism, put at least one worker on a DIFFERENT host or on cloud (T3/OpenRouter).
- **Cheap fix that works:** route `coder` to a FREE cloud coding model (e.g. `openrouter/qwen/qwen3-coder:free`, ~$0) so it never touches the local VRAM pool. Then `researcher`/`reviewer` on local M2 run parallel with `coder` on cloud.
- Before assigning a local model, **verify it's actually loaded**: `curl -s http://<host>:11434/api/tags` — do not trust memory of what "should" be there. (A model can be pulled-but-unloaded; a profile pointing at an unloaded model just stalls.)
- **Lab topology is set by the USER, not by skill text.** The user is the source of truth on which model lives on which machine. A skill (e.g. `ornith-agent`) may carry a STALE claim like "ornith:9b does NOT run anywhere" — the user confirmed ornith:9b IS live on M1 (via `ollama list` on M1, 2026-07-19). When a skill's claim contradicts the user's stated topology, the skill is wrong, not the user. Do NOT re-scan the network and override the user's word with a stale skill line. See pitfall "Don't re-probe to override the user's topology" below.
- Note: `ornith:35b-q4_K_M` is also live (on M2, as the Etsy listing-copywriter). ornith:9b on M1 is a valid candidate worker host but M1's Windows Ollama is localhost-only from WSL unless `OLLAMA_HOST=0.0.0.0` is set on M1.

### Dashboard
The `dashboard:` block in config.yaml enables the local UI. OAuth (`oauth.client_id`, `public_url`) is ONLY needed to expose it publicly — local viewing works with those empty. If the user wants to watch from another machine/phone, that's when to fill OAuth in.

## The anti-temptation rules

Your job description says "route, don't execute." The rules that enforce that:

- **Do not execute the work yourself.** Your restricted toolset usually doesn't even include terminal/file/code/web for implementation. If you find yourself "just fixing this quickly" — stop and create a task for the right specialist.
- **For any concrete task, create a Kanban task and assign it.** Every single time.
- **Split multi-lane requests before creating cards.** A user prompt can contain several independent workstreams. Extract those lanes first, then create one card per lane instead of bundling unrelated work into a single implementer card.
- **Run independent lanes in parallel.** If two cards do not need each other's output, leave them unlinked so the dispatcher can fan them out. Link only true data dependencies.
- **Never create dependent work as independent ready cards.** If a card must wait for another card, pass `parents=[...]` in the original `kanban_create` call. Do not create it first and link it later, and do not rely on prose like "wait for T1" inside the body.
- **If no specialist fits the available profiles, ask the user which profile to create or which existing profile to use.** Do not invent profile names; the dispatcher will silently drop unknown assignees.
- **Decompose, route, and summarize — that's the whole job.**

## Decomposition playbook

### Step 1 — Understand the goal

Ask clarifying questions if the goal is ambiguous. Cheap to ask; expensive to spawn the wrong fleet.

### Step 2 — Sketch the task graph

Before creating anything, draft the graph out loud (in your response to the user). Treat every concrete workstream as a candidate card:

1. Extract the lanes from the request.
2. Map each lane to one of the profiles you discovered in Step 0. If a lane doesn't fit any existing profile, ask the user which to use or create.
3. Decide whether each lane is independent or gated by another lane.
4. Create independent lanes as parallel cards with no parent links.
5. Create synthesis/review/integration cards with parent links to the lanes they depend on. A child created with unfinished parents starts in `todo`; the dispatcher promotes it to `ready` only after every parent is done.

Examples of prompts that should fan out (using placeholder profile names — substitute whatever exists on the user's setup):

- "Build an app" → one card to a design-oriented profile for product/UI direction, one or two cards to engineering profiles for implementation, plus a later integration/review card if the user has a reviewer profile.
- "Fix blockers and check model variants" → one implementation card for the blocker fixes plus one discovery/research card for config/source verification. A final reviewer card can depend on both.
- "Research docs and implement" → a docs-research card can run in parallel with a codebase-discovery card; implementation waits only if it truly needs those findings.
- "Analyze this screenshot and find the related code" → one card to a vision-capable profile for the visual analysis while another searches the codebase.

Words like "also," "finally," or "and" do not automatically imply a dependency. They often mean "make sure this is covered before reporting back." Only link tasks when one card cannot start until another card's output exists.

Show the graph to the user before creating cards. Let them correct it — including which actual profile name should own each lane.

### Step 3 — Create tasks and link

Use the profile names from Step 0. The example below uses placeholders `<profile-A>`, `<profile-B>`, `<profile-C>` — replace them with what the user actually has.

```python
t1 = kanban_create(
    title="research: Postgres cost vs current",
    assignee="<profile-A>",  # whichever profile handles research on this setup
    body="Compare estimated infrastructure costs, migration costs, and ongoing ops costs over a 3-year window. Sources: AWS/GCP pricing, team time estimates, current Postgres bills from peers.",
    tenant=os.environ.get("HERMES_TENANT"),
)["task_id"]

t2 = kanban_create(
    title="research: Postgres performance vs current",
    assignee="<profile-A>",  # same profile, run in parallel
    body="Compare query latency, throughput, and scaling characteristics at our expected data volume (~500GB, 10k QPS peak). Sources: benchmark papers, public case studies, pgbench results if easy.",
)["task_id"]

t3 = kanban_create(
    title="synthesize migration recommendation",
    assignee="<profile-B>",  # whichever profile does synthesis/analysis
    body="Read the findings from T1 (cost) and T2 (performance). Produce a 1-page recommendation with explicit trade-offs and a go/no-go call.",
    parents=[t1, t2],
)["task_id"]

t4 = kanban_create(
    title="draft decision memo",
    assignee="<profile-C>",  # whichever profile drafts user-facing prose
    body="Turn the analyst's recommendation into a 2-page memo for the CTO. Match the tone of previous decision memos in the team's knowledge base.",
    parents=[t3],
)["task_id"]
```

`parents=[...]` gates promotion — children stay in `todo` until every parent reaches `done`, then auto-promote to `ready`. No manual coordination needed; the dispatcher and dependency engine handle it.

If the task graph has dependencies, create the parent cards first, capture their returned ids, and include those ids in the child card's `parents` list during the child `kanban_create` call. Avoid creating all cards in parallel and linking them afterward; that creates a window where the dispatcher can claim a child before its inputs exist.

### Step 4 — Complete your own task

If you were spawned as a task yourself (e.g. a planner profile was assigned `T0: "investigate Postgres migration"`), mark it done with a summary of what you created:

```python
kanban_complete(
    summary="decomposed into T1-T4: 2 research lanes in parallel, 1 synthesis on their outputs, 1 prose draft on the recommendation",
    metadata={
        "task_graph": {
            "T1": {"assignee": "<profile-A>", "parents": []},
            "T2": {"assignee": "<profile-A>", "parents": []},
            "T3": {"assignee": "<profile-B>", "parents": ["T1", "T2"]},
            "T4": {"assignee": "<profile-C>", "parents": ["T3"]},
        },
    },
)
```

### Step 5 — Report back to the user

Tell them what you created in plain prose, naming the actual profiles you used:

> I've queued 4 tasks:
> - **T1** (`<profile-A>`): cost comparison
> - **T2** (`<profile-A>`): performance comparison, in parallel with T1
> - **T3** (`<profile-B>`): synthesizes T1 + T2 into a recommendation
> - **T4** (`<profile-C>`): turns T3 into a CTO memo
>
> The dispatcher will pick up T1 and T2 now. T3 starts when both finish. You'll get a gateway ping when T4 completes. Use the dashboard or `hermes kanban tail <id>` to follow along.

## Staged multi-phase builds (serialize workers that share files)

When several cards' workers all edit the SAME file(s), running them in parallel
clobbers each other (last-writer-wins). This is the #1 multi-worker failure and is
easy to trigger: a gateway auto-dispatch can grab several cards at once, and if two
of them `patch`/`write` the same `app.js`/`server.py`, the second write is applied to
a stale read and one stage's work is lost. A restarted gateway re-grabs every
`todo`/`running` card at once — exactly the hazard.

**The staged recipe (one worker at a time):**
1. Map the file-touch graph. Cards that touch the same file must NOT run together.
2. For every card you want to DEFER: `hermes kanban block <id> "staging: X runs first"` —
   block works only on ACTIVE (running/todo) cards and makes the dispatcher skip it.
   Do this BEFORE killing the worker.
3. Kill the deferred card's live worker PID (`ps -ef | grep "work kanban task <id>"`,
   then `kill -9 <pid>`). Killing the PID alone is not enough — the gateway respawns it
   unless the card is blocked/archived.
4. Leave exactly ONE card running. Verify with `ps -ef | grep "hermes -p <profile>"`.
5. When it finishes, VERIFY the code actually landed (see below), then `hermes kanban
   unblock <next_id>` so the dispatcher grabs it on the next 60s tick.
6. Repeat until all stages done; then integrate + restart the service.

**reclaim ordering:** `hermes kanban reclaim <id>` only works on a RUNNING card. If you
block it first, reclaim then errors ("not running"). Block is the durable guard (dispatcher
skips blocked), so reclaim is optional — but if you want a clean claim release, reclaim
BEFORE block, not after.

**Phantom 'done' cards — verify the code landed.** A worker can mark a card `done`
(clean exit, no `kanban_complete`) or the card can show `done` while the file change
never happened. Before trusting any `done` status, grep the codebase for a feature
marker (the function name / endpoint the card promised). If nothing is found, the stage
is NOT done — re-run it or inspect the worker log.

**Recover a lost/in-compacted spec before re-asking the user.** If the user references a
spec/plan whose text you can't find (lost to a context compaction, or you "forgot" what
they meant), recover it from durable stores FIRST — do not ask them to re-paste:
- `SELECT id,title,status,assignee,body FROM tasks ORDER BY created_at DESC` in
  `~/.agent-home/kanban.db` — decomposed spec stages often live as card titles/bodies
  (e.g. an "MCv3 S1..S5" set is exactly how a vanished tiered spec was recovered).
- `ls -t ~/.agent-home/pastes/*.txt` then grep them for a distinctive phrase.
- `search_files` the project tree + `~/.agent-home/logs` for a keyword from the spec.
Only ask the user to re-share if all three come up empty.

Exact copy-paste command sequence (reclaim → block → kill worker PID, plus a
kanban.db status-poll one-liner and the verify-then-unblock loop):
`references/stage-serialization.md`.

## Common patterns

**Fan-out + fan-in (research → synthesize):** N research-style cards with no parents, one synthesis card with all of them as parents.

**Parallel implementation + validation:** one implementer card makes the change while one explorer/researcher card verifies config, docs, or source mapping. A reviewer card can depend on both. Do not make the implementer own unrelated verification just because the user mentioned both in one sentence.

**Pipeline with gates:** `planner → implementer → reviewer`. Each stage's `parents=[previous_task]`. Reviewer blocks or completes; if reviewer blocks, the operator unblocks with feedback and respawns.

**Same-profile queue:** N tasks, all assigned to the same profile, no dependencies between them. Dispatcher serializes — that profile processes them in priority order, accumulating experience in its own memory.

**Human-in-the-loop:** Any task can `kanban_block()` to wait for input. Dispatcher respawns after `/unblock`. The comment thread carries the full context.

## Pitfalls

**Don't re-probe the network to override the user's stated lab topology.** If the user says "M1 has ornith:9b" or "M2 is the local-Ollama box," treat that as ground truth. Scanning the subnet and concluding "no ornith found, so you're wrong" is a failure mode that burned a real session — the scan may hit the wrong IP (WSL NAT hides the Windows host LAN IP), or a model may be on a different port/host than the scan covered. When a live probe appears to contradict the user, the probe/assumption is wrong, not the user. Verify by asking the user for the exact endpoint rather than declaring the user mistaken. (This lab: M1 runs Hermes + T3 and hosts ornith:9b on Windows Ollama; M2 = gpu-node-2 local Ollama; M3 = aux-node tiny qwen3.5. These are user-stated and authoritative. NOTE: M3 moved <lan-ip> → aux-node on 2026-08-04.)

**Inventing profile names that don't exist.** The dispatcher silently fails to spawn unknown assignees — the card just sits in `ready` forever. Always assign to a profile from your Step 0 discovery; ask the user if you're unsure.

**Bundling independent lanes into one card.** If the user asks for two independent outcomes, create two cards. Example: "fix blockers and check model variants" is not one fixer task; create a fixer/engineer card for the fixes and an explorer/researcher card for the variant check, then optionally gate review on both.

**Over-linking because of wording.** "Finally check X" may still be parallel with implementation if X is static config, docs, or source discovery. Link it after implementation only when the check depends on the implementation result.

**Forgetting dependency links.** If the task graph says `research -> implement -> review`, do not create all tasks as independent ready cards. Use parent links so implement/review cannot run before their inputs exist.

**Reassignment vs. new task.** If a reviewer blocks with "needs changes," create a NEW task linked from the reviewer's task — don't re-run the same task with a stern look. The new task is assigned to the original implementer profile.

**Argument order for links.** `kanban_link(parent_id=..., child_id=...)` — parent first. Mixing them up demotes the wrong task to `todo`.

**Don't pre-create the whole graph if the shape depends on intermediate findings.** If T3's structure depends on what T1 and T2 find, let T3 exist as a "synthesize findings" task whose own first step is to read parent handoffs and plan the rest. Orchestrators can spawn orchestrators.

**Tenant inheritance.** If `HERMES_TENANT` is set in your env, pass `tenant=os.environ.get("HERMES_TENANT")` on every `kanban_create` call so child tasks stay in the same namespace.

## Recovering stuck workers

When a worker profile keeps crashing, hallucinating, or getting blocked by its own mistakes (usually: wrong model, missing skill, broken credential), the kanban dashboard flags the task with a ⚠ badge and opens a **Recovery** section in the drawer. Three primary actions:

1. **Reclaim** (or `hermes kanban reclaim <task_id>`) — abort the running worker immediately and reset the task to `ready`. The existing claim TTL is ~15 min; this is the fast path out.
2. **Reassign** (or `hermes kanban reassign <task_id> <new-profile> --reclaim`) — switch the task to a different profile (one that exists on this setup) and let the dispatcher pick it up with a fresh worker.
3. **Change profile model** — the dashboard prints a copy-paste hint for `hermes -p <profile> model` since profile config lives on disk; edit it in a terminal, then Reclaim to retry with the new model.

Hallucination warnings appear on tasks where a worker's `kanban_complete(created_cards=[...])` claim included card ids that don't exist or weren't created by the worker's profile (the gate blocks the completion), or where the free-form summary references `t_<hex>` ids that don't resolve (advisory prose scan, non-blocking). Both produce audit events that persist even after recovery actions — the trail stays for debugging.

## PITFALL — orphaned worker holds M2 while its board is empty/corrupted (2026-08-22)
A worker can be **alive** (`ps` shows it at 40% CPU) but its task `t_xxx` no longer exists in
any readable `board.json` (the board got corrupted/emptied — `overnight-build` showed 0 cards).
In that state the dashboard shows NO ⚠ badge (no card to flag), so the standard Recovery
actions (reclaim/reassign) can't target it — but the process is STILL holding the profile's
model slot on M2, which blocks other workloads (e.g. TikTok's 11:30 post hit a 404 because the
slot was occupied). Symptom the user sees: "everything's stuck, nothing fired, but no error."

**Detection (not via the dashboard — it's blind here):**
```bash
ps aux | grep -E "work kanban task t_" | grep -v grep   # any live worker with an orphaned task
```
Cross-check the task id against the boards: if `grep -r "<task_id>" ~/.agent-home/kanban/boards/`
finds nothing, it's orphaned.

**Recovery when orphaned (dashboard can't help):**
1. `kill -9 <pid>` — free the held model slot / compute immediately. The gateway may respawn
   it, but with no card to claim it exits (or respawns into the empty board and idles).
2. `hermes kanban reclaim <task_id>` as a belt-and-braces reset even if the board looks empty.
3. If boards are corrupted (multiple `board.json` missing/empty), reset the active board to
   clean so the decomposer has somewhere valid to write: the `current` pointer
   (`~/.agent-home/kanban/current`) names the active board; ensure that board's `board.json` exists
   and is valid JSON (`{}` with an empty `cards` list is fine — NOT a missing file).

**Lesson:** a wedged worker is an outage even without a ⚠ badge. The self-heal cron must check
*live `ps` state*, not just kanban flags, and `kill -9` orphaned workers holding shared
resources (M2 model slot). See hermes-cron-management "self-heal checks flags not reality".
