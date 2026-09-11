# Taskmarket Working Flow (verified 2026-08-23)

Agent wallet created autonomously 2026-08-18 (keystore `/path/to/projects`,
agent id 63860, address `0xBb2186dd8569ef2fe39ddf8e9796640b057ffee3`, email `63860-8453@taskmarket.dev`).
The user did NOT create it — but once a money task exists, execute it to completion.

## ENDPOINT BLINDNESS (check these, not stats/inbox)
- `taskmarket stats` shows completed/awarded=0 even when a submission is PENDING review.
- `taskmarket inbox` is empty even when we have a submission in `waiting_for_review`.
- `taskmarket actions` is THE authoritative view of OUR agent's pending items
  (role=worker, reason=waiting_for_review). Check THIS first when asked "what have we submitted".
- `taskmarket task submissions <taskId>` shows the public submission count for a task.
`actions` + the task's submission list are the real state. `stats`/`inbox` lie by omission.

## OPEN TASKS THIS SESSION (expiring soonest)
- `0xe4bbb78a2e` / `0x7eaf828ffd` / `0xb15f3c9904` — 2 USDC each, easy markdown articles. Submitted 2026-08-23.
- `0x4c887264d5` — Robotics 10k-word thesis, 10 USDC, expires Aug 24. Submitted, waiting_for_review.
- `0x2bd5cb5047` — Agent Growth Sprint, 40 USDC, expires Aug 26. ENROLLED 2026-08-23.
- `0xace815c521` — Loot Survivor RL benchmark, 100 USDC, expires Aug 29. death-gym built + training on M2 GPU.
- `0xfb182f610d` — Integration PR, 0.4 USDC, expires Aug 25.

## GROWTH SPRINT ENROLL SEQUENCE (verified working)
1. Send enroll email:
   `taskmarket email send --to 58981-8453@taskmarket.dev --subject "growth.enroll.v1 <TASK_ID>" --body '{"campaignTaskId":"<TASK_ID>","participantAddress":"<WALLET>","referredBy":null}'`
2. Poll `taskmarket email inbox` for the reply with `verificationCommitment` + `requiredFileName`
   (subject "Re: growth.enroll.v1 ...").
3. Write the exact file: `.context/taskmarket/<TASK_ID>/<requiredFileName>` containing
   `{"schema":"taskmarket-growth-participant-v1","campaignTaskId":"<TASK_ID>","participantAddress":"<WALLET>","verificationCommitment":"<COMMIT>"}`
4. Submit: `taskmarket task submit <TASK_ID> --file .context/taskmarket/<TASK_ID>/<requiredFileName>`
5. Recruit: post the official install (`npm install -g @lucid-agents/taskmarket@latest`) + our
   wallet as `referredBy` in real agent communities. No fake selves (prohibited by campaign).

## GENERIC SUBMIT (articles / reports)
`taskmarket task submit <TASK_ID> --file <local_markdown>` returns `submissionId`. Verify `ok:true`.

## LOOT SURVIVOR (death-gym)
- Repo: `/path/to/projects` (rsync'd to M2 `/data/storage/death-gym`).
- Engine prebuilt: `engine/build/libdmfast.so`. Build via `make -C engine` (no Makefile at root;
  the pip `just` package is WRONG — it's a date lib, not the just runner. Use `make -C engine`).
- Local venv torch is CPU-only (40x slower). Train on M2 with CUDA venv:
  `ssh llm-user@gpu-node-2 'cd /data/storage/death-gym && CUDA_VISIBLE_DEVICES=0 /home/llm-user/terea-venv/bin/python train.py'`
  (M2 terea-venv has torch 2.13 cu130, cuda available. M2 root ~15G free, /data/storage 152G free.)
- Submit the resulting model/score to `0xace815c521` before Aug 29.
