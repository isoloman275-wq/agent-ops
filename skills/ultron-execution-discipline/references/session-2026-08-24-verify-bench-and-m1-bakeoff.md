# Session learnings — 2026-08-24 (verify-bench, swarm wiring, M1 bake-off)

## 1. Independent verifier > self-critique (MEASURED)
Battery: 20 verifiable tasks, deterministic Python oracle (`verify-bench/verify_bench.py`),
worker qwen3.5:2b-aux (M3), verifier ornith:9b (M1).
- Bare worker: 9/10. +self-critique: 9/10 (same error repeated, 2× latency).
- +independent verifier: caught the worker's blind-spot error (dropped "19" from primes<20)
  EVERY run. Observer independence is the mechanism — self-grading cannot see its own bias.
- Kanban wiring: `hermes kanban swarm --worker P --verifier V --synthesizer S`. Built
  `independent-verifier` profile (ornith:9b-64k @ M1) with AGENT.md protocol: re-read artifact
  with tools, gate on evidence ("Gate: PASS"), never trust worker claims.
- Verified end-to-end: worker built exact file → verifier re-read it → Gate PASS.

## 2. Profile fallback chains silently violate machine rules
Builder profile fell back to M2/qwen3.8 despite "M1-only" intent → OOM crash-loop 12× against
a machine under an explicit hands-off order. Task-level `kanban set-model --provider` did NOT
survive dispatch. Working fix: strip the forbidden provider entirely from profile config
(ollama-m2 blocks, Hemi custom_providers near line ~450+, stale base_url under `model:`),
then VERIFY routing via `profiles/<p>/logs/agent.log` lines `API call #N: model= base_url=`.
YAML-check after edits: `python3 -c "import yaml; yaml.safe_load(open(p))"`.

## 3. One GPU = one concurrent kanban worker (measured)
Two parallel workers on M1's 8GB card: artifacts written correctly each attempt, but both
crashed before protocol-complete every cycle (SIGKILL loops). Solo card: clean pass in ~7min.
Cap small-GPU nodes to 1 concurrent card; serialize worker→verifier.

## 4. M1 Ollama unload wedge — root cause + working protocol
`keep_alive:0` unload doesn't stick because abandoned requests in Ollama's queue RE-REQUEST
the model → reload-loop at `-c 131072` (128K KV on 8GB) → all generates time out.
FIX: kill llama-server AND ollama serve → restart serve → immediately load target model
`keep_alive:-1` + capped ctx (claims the slot) → verify `/api/ps` resident = target only.
Post-fix: 44.5 tok/s, ~2.3s/short call vs 240s+ wedged.

## 5. Importing a HF GGUF into Windows Ollama on M1
- Download with `curl -L -H "Authorization: Bearer $HF_TOKEN"`; on exit 92 (HTTP/2 reset)
  RESUME with `-C -` — continues cleanly to completion.
- Create model via Modelfile with `FROM C:\Users\Admin\hf_models\<file>.gguf` (local Windows
  path). `FROM sha256:<digest>` FAILS ("pull model manifest: file does not exist") even after
  a successful blob push — blob-API create path does not work for this.
- Strip CRLF from Modelfile (printf/tr, not PowerShell) before create.
- Find max GPU layers empirically per user instruction: probe num_gpu upward until OOM, back
  off one. 9B Q5_K_M on RX5700XT 8GB: num_gpu=41 fits (5.9GB), 42 OOMs.

## 6. M1 model bake-off (serial, same battery/oracle)
- ornith:9b-64k 19/20 (~1.7s/task) vs qwen38-9b (Qwen3.8-9B-Distill Q5_K_M) 18/20 (~2.2s/task).
- Failure profiles DIFFER between models → cross-model verification catches both blind spots.
- qwen38-9b leaks a "Thinking Process:" preamble — needs FINAL-extraction wrapper for agents.
- Reusable harness: `verify-bench/model_bench.py <model>`; results in `BAKEOFF.md`.

## 7. User workflow corrections this session
- Serial benching only: "they can't run simultaneously on m1" — one model at a time, one
  worker at a time.
- Max-fit layers first: "set num_gpu to max or max+1... that's how we find what fits THEN
  benchmark."
- M2 hands-off days are absolute: any route that CAN reach M2 must be structurally removed,
  not discouraged.
- Agent context floor: verifier-only models may be capped at 8K, but anything Hermes-worker-
  viable keeps ≥64K (Hermes tool-use floor).
