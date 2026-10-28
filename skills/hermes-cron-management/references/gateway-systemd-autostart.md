# Gateway auto-start via systemd-user (local WSL2 box)

Goal: keep the Hermes gateway (and its embedded cron scheduler) alive across
logins so crons don't silently skip when the desktop sleeps/reboots. Chosen
over the GPU-NODE option when the user prioritizes local latency/responsiveness.

Environment: WSL2 Ubuntu, systemd --user running, Hermes installed via pipx
(`hermes` on PATH → venv at `/path/to/projects`).

## Minimal stub unit (Hermes rewrites this — a bare stub is intentional)
`~/.config/systemd/user/hermes-gateway.service`:
```
[Unit]
Description=Hermes Agent Gateway
After=network-online.target
Wants=network-online.target
[Service]
Type=simple
ExecStart=/path/to/projects -m hermes_cli.main gateway run --replace
Restart=always
RestartSec=3
[Install]
WantedBy=default.target
```

## Command transcript (verified 2026-07-16)
```bash
# 1. confirm systemd --user alive
systemctl --user status | head -3

# 2. daemon-reload picks up the new unit
systemctl --user daemon-reload

# 3. stop the manually-launched gateway so systemd owns the socket
ps aux | grep "hermes.*gateway run" | grep -v grep   # note PID (e.g. 118)
kill 118
sleep 2

# 4. enable + start (auto-starts on every WSL login)
systemctl --user enable --now hermes-gateway.service

# 5. verify
systemctl --user status hermes-gateway.service --no-pager | grep -E "Active:|Main PID"
# -> active (running), Main PID <new>
```

## What Hermes rewrites the unit into (post-launch, under systemd)
Hermes detects `under_systemd=yes` and overwrites the file. The rewritten form
ADDS env your jobs need — copy this verbatim if you must hand-write instead of
letting Hermes self-manage:
```
[Unit]
Description=Hermes Agent Gateway - Messaging Platform Integration
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=/path/to/projects -m hermes_cli.main gateway run --replace
WorkingDirectory=/path/to/projects
Environment="PATH=/path/to/projects
Environment="VIRTUAL_ENV=/path/to/projects
Environment="HERMES_HOME=~/.agent-home"
Restart=always
RestartSec=5
RestartMaxDelaySec=300
RestartSteps=5
RestartForceExitStatus=75
KillMode=mixed
KillSignal=SIGTERM
ExecReload=/bin/kill -USR1 $MAINPID
TimeoutStopSec=210
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```
CRITICAL: the `PATH` includes `<windows-path>` and `HERMES_HOME` is set.
Without these, crons that call `powershell.exe` / `cmd.exe` (the video pipeline,
YouTube cross-post) fail even though the gateway is "up".

## Self-healing verification
```bash
kill $(pgrep -f "hermes_cli.main gateway")   # systemd respawns
sleep 6
systemctl --user status hermes-gateway.service --no-pager | grep -E "Active:|Main PID"
ps aux | grep "hermes_cli.main gateway" | grep -v grep   # fresh PID + child
```
NOTE: the first respawn after a kill may exit 1 once (socket in TIME_WAIT).
systemd retries after RestartSec=5 and comes up clean. Wait ~6s before judging.

## Caveat
Auto-start on WSL login still does nothing if Windows is fully shut down / asleep
— WSL isn't running, so cron can't fire until you log in. For true 24/7 immunity
use the GPU-NODE (gpu-node) gateway option instead.
