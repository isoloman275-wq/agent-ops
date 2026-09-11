# Loot Survivor (death-gym) — M2 GPU Build Recipe

The rl-competition-playbook inside taskmarket-worker defaults to CPU training.
the operator explicitly wants the Loot Survivor run on M2 GPU for speed (~70x: 460 fps
CPU → ~33,000 fps GPU on the dual RTX 3060). When he directs M2, follow this.

## Build the CUDA venv on M2 (once)
```bash
ssh llm-user@gpu-node-2
cd /data/storage/death-gym
python3 -m venv .venv
source .venv/bin/activate
pip install numpy pytest safetensors
pip install torch --index-url https://download.pytorch.org/whl/cu124
make -C engine          # compile libdmfast.so
```
Verify: `python -c "import torch; print(torch.cuda.is_available())"` → True.
The repo needs `safetensors` (checkpoint.py imports it) — without it train.py
dies at import with ModuleNotFoundError.

## Before launching (single-model rule — mandatory)
- Confirm ComfyUI is NOT rendering: `curl -s 127.0.0.1:8188/queue` → empty.
- Confirm GPUs idle + no Ollama resident: `nvidia-smi` + `curl gpu-node-2:11434/api/ps`.
- Claim M2 via m2_arbiter so no cron/worker collides.

## Launch
```bash
cd /data/storage/death-gym
source .venv/bin/activate
export PYTHONPATH=/data/storage/death-gym
nohup python train.py --total-steps 20000000 --run-name loot100 \
  --checkpoint-every 20 --seed 42 > local/logs/loot100.log 2>&1 &
```
Confirm: log shows `fps:` in tens of thousands (GPU), and `nvidia-smi` VRAM climbs.

## Pitfalls (2026-08-23)
- Don't assert "no venv / no setup" from one path. Check all disks + running
  processes before concluding. A single empty result is NOT proof of absence.
- SSH nested quoting: use single quotes for inner `python -c "..."` to avoid EOF
  errors under `ssh bash -lc`.
- Do NOT fall back to CPU on M1 after he vetoes it, and do NOT re-launch after a
  "no". Halt until explicit go. M2 HARD RULE: this runs ONLY because the operator explicitly
  directed it on an idle M2 — never on your own initiative.
