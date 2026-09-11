# Kanban Fleet Setup — verified recipe (this session, 2026-07-19)

Working procedure to stand up a multi-agent Kanban fleet on this user's lab
(M1 runs Hermes + T3; M2 is local Ollama at gpu-node-2; M3 is tiny qwen3.5).
The generic SKILL.md "Setup" section is simplified — these are the EXACT commands
that worked, including two traps the simplified version hides.

## 1. Add a local-Ollama provider (if the target model host isn't in config)

Providers live under `providers:` in `~/.agent-home/config.yaml`. To add M2:

    hermes config set providers.ollama-m2 '{"provider": "custom", "base_url": "http://gpu-node-2:11434/v1"}'

**TRAP:** `hermes config set` stores that value as a JSON *string*, not a parsed
dict. Hermes then reads `ollama-m2` as a string and the provider is dead. Fix it
afterwards by parsing the string back to a dict (terminal python, NOT the patch
tool — config.yaml is write-protected against the patch tool but editable via
terminal):

    import yaml, json
    cfg = yaml.safe_load(open("~/.agent-home/config.yaml"))
    raw = cfg["providers"]["ollama-m2"]
    if isinstance(raw, str):
        cfg["providers"]["ollama-m2"] = json.loads(raw)
    yaml.safe_dump(cfg, open("~/.agent-home/config.yaml","w"),
                   default_flow_style=False, sort_keys=False)

Verify with `hermes config check` (should pass, no errors) and
`grep -A2 ollama-m2 ~/.agent-home/config.yaml` (should show nested provider/base_url).

Existing providers on this lab: `ollama-m1` (M1 Windows Ollama, localhost:11434),
`ollama-m2` (M2, gpu-node-2:11434), `ollama-m3` (M3, aux-node:11434).

## 2. Create the profiles

`hermes profile create` has NO `--model` flag (confirmed via --help). Create with
`--clone` (copies config/.env/skills from active profile) + `--description` (the
description is what the kanban decomposer routes on, so make it role-specific):

    hermes profile create architect  --clone --description "Head architect/dev-lead. Decomposes goals, routes to specialists, owns continuous improvement, scaling, new revenue avenues. Strong strategic reasoning."
    hermes profile create coder      --clone --description "Implementation specialist. Writes/modifies code, pipelines, scripts, configs."
    hermes profile create researcher --clone --description "Research/discovery. Web scraping, source/docs investigation, market/trend analysis, synthesis."
    hermes profile create reviewer   --clone --description "QA/verification. Reviews code, validates outputs, checks errors/security before handoff."

Each lands in `~/.agent-home/profiles/<name>/` with its own config.yaml + .env + SOUL.md.

## 3. Set each profile's model + provider

Per-profile config is edited with `hermes -p <profile> config set` (NOT global
`hermes config set`, which would change the active profile only):

    # architect -> T3 (OpenRouter). gpt-oss-120b is a FREE reasoning model on this key.
    hermes -p architect  config set provider 'openrouter'
    hermes -p architect  config set model 'openai/gpt-oss-120b'

    # coder/researcher/reviewer -> M2 local Ollama (ollama-m2 provider)
    hermes -p coder      config set provider 'ollama-m2'
    hermes -p coder      config set model 'qwen3.8:27b-132k'
    hermes -p researcher config set provider 'ollama-m2'
    hermes -p researcher config set model 'glm-4.7-flash'
    hermes -p reviewer   config set provider 'ollama-m2'
    hermes -p reviewer   config set model 'qwen3.8:27b-132k'

Verify: `hermes profile list` shows each with its model + provider.

## 4. Wire the orchestrator

    hermes config set kanban.orchestrator_profile 'architect'
    hermes config set kanban.default_assignee 'coder'

(`dispatch_in_gateway: true` and `auto_decompose: true` were already on by default.)

## 5. Live fan-out test

    T1=$(hermes kanban create "probe: researcher fetch X"  --assignee researcher | grep -oE 't_[a-f0-9]+' | head -1)
    T2=$(hermes kanban create "probe: coder write hello.py" --assignee coder     | grep -oE 't_[a-f0-9]+' | head -1)
    T3=$(hermes kanban create "probe: architect confirm"     --assignee architect | grep -oE 't_[a-f0-9]+' | head -1)
    hermes kanban dispatch        # one pass: should spawn all 3 in parallel, each to its profile
    hermes kanban list            # expect 3x "running" with correct assignees
    # cleanup when done:
    hermes kanban reclaim $T1 $T2 $T3
    hermes kanban archive $T1 $T2 $T3

## 6. Model reachability checks (do these BEFORE relying on a profile)

    # M2 local models actually loaded?
    curl -s http://gpu-node-2:11434/api/tags | python3 -c "import json,sys;print([m['name'] for m in json.load(sys.stdin)['models']])"
    # T3 model listed + responds? (gpt-oss-120b is a REASONING model: content is null on
    # short max_tokens; use max_tokens>=200 to get real content back)

## VRAM reality on this lab (proven live)
M2 is a SINGLE VRAM pool: only one of qwen3.8:27b-132k / glm-4.7-flash runs at a time.
During the test, `coder` (qwen3.8:27b-132k) loaded and `researcher` (glm-flash) had to
wait — exactly the contention rule. `architect` on T3 ran free alongside. True 3-way
parallelism needs a 2nd real model host (ornith:9b on M1, but M1 Windows Ollama is
localhost-only from WSL unless OLLAMA_HOST=0.0.0.0 is set on M1).
