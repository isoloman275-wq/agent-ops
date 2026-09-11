# Loot Survivor / RL-competition verified runbook (2026-08-27)

Session-derived facts from Taskmarket TSK-E4RXQS7X ($100 USDC). Repo on M2 at
`/data/storage/death-gym`. Companion detail for RULE 15/16/17 in SKILL.md.

## Score model & leaderboard pacing

- Score = mean XP over 3 banks x 16,384 worlds, public seeds 3930/7717/20477;
  prize decided later on PRIVATE seeds.
- Public high moved Aug 22->27: 276 -> 304 -> 305.2 -> **327.7** then FLAT ~2 days.
  Re-fetch the raw leaderboard gist before every decision; never decide from memory.
- Training-time `death_xp` OVERSTATES eval by ~15-40% (eval pins seeds, kills
  curriculum). Planning heuristic on sane configs: eval ≈ death_xp - 40..60.

## Config results table (MLP dm_lstm_v1, Loot Survivor)

| Config | Steps | Training metric | Eval |
|---|---|---|---|
| Resumed stripped-reward base + invented overrides | 2B | avg_xp ~145 | 144.7 |
| Same + --armor-coef 1.0 (nukes ALL rewards) | 2B | death_xp 75 | 96.9 |
| **From scratch, engine DEFAULT_PARAMS restored** | 2B | avg_xp 354 | **299.0** |
| Defaults + deep-weight 2.0 + sil-coef 0.5 + hot curriculum | 2B | death_xp 353 | 225.6 |
| Defaults + buy-boost 3x/30% ramped | 2B | avg_xp 344 | 295.4 |
| Defaults + buy-boost 5x/40% ramped | **4B** | death_xp 370 | **308.0** |

Lessons: engine defaults beat every hand-rolled override; stacking extra
techniques HURT; buy-boost neutral at 2B, real gain came from 4B steps. Bigger
models hurt (512/512/3 = 167.6 vs baseline 181).

## Competition eligibility (hard rejects)

- Allowed archs: dm_lstm_v1 / dm_mlp_v1 / dm_transformer_v1; d_model {128,256},
  n_heads {4,8}, embed/hidden {128,256,512}.
- `--equivariant-head` and `--quantile-value` are INELIGIBLE (change tensor set).
  train.py prints NOT SUBMITTABLE and means it — do not burn a lane on them.
- Default transformer needs ~11.4 GiB → OOMs on a 12 GB card unless GPU fully
  empty (kill ComfyUI python :8188 + unload Ollama llama-server first);
  trained at 8.9 GiB d_model=128/n_layers=2/n_heads=4 with
  PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True. fps ~2.5k vs MLP ~35k.

## Resume/continuation traps (both bit us)

1. `--total-steps` is GRAND TOTAL, not additional. Resuming a 61,035-iter
   checkpoint with `--total-steps 2000000000` = 0 iterations ("Training done:
   ...in 0.0s", junk final.safetensors). Continue via raising total past the
   resume point, or `--reset-iteration --fresh-optimizer` with a normal total
   (reset variant also restarts LR schedule cleanly).
2. Rotating/deleting a training log while its process runs leaves it writing to
   a deleted inode: log goes stale, training continues. Ground truth = PID alive
   + checkpoint-dir mtimes + nvidia-smi util, NEVER log recency alone.

## SWA + temperature — NEGATIVE result

Leaderboard winner metadata revealed "SWA over newest-5 checkpoints +
policy-logit temperature tau=3.0". Measured on our lootR3 ckpts: SWA alone
284.0 (vs 299 final); SWA+tau=3.0 = 189.2 catastrophic. Their post-processing
is regime-specific. If retried, tune tau on HELD-OUT seeds (101/4242/90210).
Impl notes: safetensors carries arch/meta as file-level METADATA
(save_file(..., metadata=final_meta)); load_file() drops it. Policy class =
LSTMPolicy (no memory_type kwarg). Use competition_score(), not eval_policy()
(seed=None TypeError).

## Untried eligible levers (researched, ranked)

1. `--ei` expert iteration (+13 xp/decision measured by repo author; slow).
2. `--sil-coef 0.1` small self-imitation (0.5 was too hot).
3. Negative `--deep-weight -0.5` anti-forgetting on early game.
4. `--world-pool` / `--group-size` variance reduction (43% xp variance is start-state).
5. Widen shop_gate_mask directly in train.py (default only L6-11 / gold>=24).
   Bare `--explore-eps 0.05` lane CRASHED (CUDA device-side assert in
   collect_rollout forced-exploration path) — treat flag as unstable here.

## Submission timing (the operator directive, 2026-08-27 — refines RULE 15)

Never submit early just because a score beats the current high. Hold the best
artifact until near-deadline; submit once at the last responsible moment after
re-checking the live leaderboard. Scores keep improving while lanes run.

## Ops hygiene that worked

- Lane-per-GPU via `ssh -f host '/tmp/laneX.sh'`; watcher polls parent PID with
  `kill -0` then auto-fires eval + next lane. Chain carried 3 lanes over days.
- Immediately eval each finished lane (`train.py --resume <final>
  --competition-eval --shop-gate`) so scores accumulate while other lanes run.
