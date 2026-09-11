# Verified Claims Registry

> Auto-generated probe results. Each entry records a factual claim, the
> verification probe that proved or disproved it, and the evidence.
>
> **Rule**: an empty probe = "not found at THIS place" — NEVER "doesn't exist".

## 2026-08-24 — Engine smoke test (post-bugfix validation)

### M2 State (gpu-node-2)
| Claim | Probe | Verdict | Evidence |
|---|---|---|---|
| M2 process state | SSH→ps aux + nvidia-smi + Ollama /api/ps | ✓ VERIFIED | No training processes. Ollama: NO_MODELS_LOADED. GPU: 2×RTX3060, ~3% util each, ~3GB VRAM used per card. |
| qwen3.8:27b-132k loaded on M2 | curl :11434/api/ps | ✗ FAILED | NOT loaded (model was unloaded for testing). Probe correctly detected absence. |
| ComfyUI running on :8188 | curl :8188/queue | ✗ FAILED | Unreachable from sandbox — likely bound to M2 localhost. Expected in this environment. |

### M3 State (aux-node)
| Claim | Probe | Verdict | Evidence |
|---|---|---|---|
| M3 process state | curl :11434/api/ps | ✓ VERIFIED | NO_MODELS_LOADED. Node reachable. |
| qwen3.5:2b-aux NOT loaded on M3 | curl :11434/api/ps | ✓ VERIFIED (negation) | Confirmed absent. Negation logic correctly inverted the probe result. |

### Training State
| Claim | Probe | Verdict | Evidence |
|---|---|---|---|
| Training is dead | SSH→ps aux on M2 | ✓ VERIFIED | No training/python processes found on M2. |

### Test Matrix
| Test | Input | Expected Verdict | Actual | Pass? |
|---|---|---|---|---|
| Multi-claim + negation | "qwen3.5 is not loaded… qwen3.8 is loaded… ComfyUI is running… Training is dead" | FAIL (qwen3.8 + ComfyUI wrong) | FAIL | ✓ |
| No claims | "Let me go do some work" | CLEAN | CLEAN | ✓ |
| Negation only (no probes) | "qwen3.5 is NOT loaded" | UNVERIFIED | UNVERIFIED | ✓ |
| Positive only (no probes) | "qwen3.8 is loaded on M2" | UNVERIFIED | UNVERIFIED | ✓ |
| Syntax validation | ast.parse(verify_guard.py) | No errors | SYNTAX OK | ✓ |

## 2026-08-21 — Prior baseline (pre-bugfix, partial)
| Claim | Probe | Verdict | Evidence |
|---|---|---|---|
| M2 GPU state | nvidia-smi via SSH | ✓ | 2×RTX3060, GPU0 ~10.6GB/12GB used (ComfyUI render), GPU1 idle |
| M3 Ollama | curl :11434/api/ps | ✓ | Holds qwen3.5:2b-aux + qwen3.5:2b-q4_K_M |
| Te reo base model | ls on M2 /data/storage | ✓ | 15GB qwen25-7b shards present (4 safetensors + config/tokenizer) |

---

**How to use this file:** After any significant lab action, append a new section with date/time + probe results. Reference this file when debugging — it's ground truth for what was verified when. Do NOT use stale entries to override fresh probes; always re-verify before claiming current state.