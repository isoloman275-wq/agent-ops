# Agent Ops

**14 operating-discipline skills for autonomous AI agents** — verification
layers, crash-loop forensics, multi-agent orchestration, memory curation, and
session hygiene. The difference between an agent that demos well and an agent
that runs unattended for months.

Distilled from a production agent fleet: these are the rules that were each
paid for with a real failure first.

## What's inside

### Truth layer
| Skill | What it teaches |
|---|---|
| `verify-before-claim` | Every factual claim needs same-turn tool evidence |
| `evidence-before-claim` | Behavioral discipline: what humans must verify |
| `ultron-execution-discipline` | Execute-don't-narrate: irreversible-action gates, watcher patterns |

### Reliability
| Skill | What it teaches |
|---|---|
| `hermes-crash-breaker` | Crash-loop root-cause diagnosis (hint: it's usually config, not code) |
| `hermes-cron-management` | Scheduled-job patterns that don't silently die |
| `agent-verification-harness` | Independent verifier swarms with real A/B benchmark data |
| `hermes-process-reaper` | Wedged-process cleanup without killing live work |

### Orchestration
| Skill | What it teaches |
|---|---|
| `kanban-orchestrator` | Decompose work for a multi-agent board |
| `kanban-worker` | Worker-side patterns and edge cases |
| `fleet-synthesis-orchestration` | Merge parallel agent outputs into one deliverable |

### Memory & hygiene
| Skill | What it teaches |
|---|---|
| `self-editing-memory` | Letta-pattern self-curated memory blocks (tool included) |
| `skills-hub-install` | Acquire skills from public hubs safely |
| `hermes-agent-skill-authoring` | Write skills that are replicable, not lab-specific |
| `session-librarian` | Session archive organization |
| `weekly-review-planning` | Weekly reset: commitments, stalled work, next plan |

## Design principles

- **Rules with receipts** — each discipline includes the failure that motivated it.
- **Tools included** — self-editing memory ships with its working script.
- **Agent-agnostic** — applies to any autonomous agent (Hermes, Claude Code, OpenClaw, custom).

## License

MIT.
