# Lab Dashboard & Cron Visibility — reusable techniques

Two small but repeatedly-missed techniques that bit the lab on 2026-08-23
(MC/Dashboard showed blank; 2-min nudge was invisible). Capture them so the
next session starts knowing.

## 1. Serve local dashboards over HTTP, never file://

A browser tab opened from `file:///home/.../dashboard.html` cannot issue a browser HTTP request to
local JSON file — the browser blocks `file://` to `file://` XHR/fetch as a CORS
violation. Result: the page renders its static shell (stream cards, layout) but
every browser request for '/home/.../status.json' silently fails, so GPU/kanban/crons/skills
stay blank and the dashboard looks "dead / not live".

FIX (verified working):
- Start a static server from the dashboard dir:
  `cd ~/.agent-home/dashboard && python3 -m http.server 8788 --bind 127.0.0.1`
  (run as a long-lived background process; it never exits).
- Point every HTML page's browser HTTP requests to `http://127.0.0.1:8788/status.json`
  (use sed to swap the `file://...status.json` path for the HTTP URL across
  all dashboard HTML files at once).
- Open the page in preview as `http://127.0.0.1:8788/mission-control-v2.html`
  (NOT file://). Now `fetch` succeeds and live data populates.

The collector (`dashboard_collector.py`, cron `*/2 * * * *`) writes
`status.json` from live sources (GPU via ssh nvidia-smi, TikTok
content_history.json post count, kanban, crons, AUX-NODE api/ps, skills dir). The
server just exposes it. Keep both: collector (data) + http.server (delivery).

## 2. Cron nudges must deliver where the user will see them

A self-nudge / watchdog cron with `deliver: local` writes its output into the
agent session and NOTHING reaches the user — they never see it, so "it does
nothing" from their side even though it fired every 2 min (last_run_at proves
it ran). The nudge is useless if invisible.

FIX (verified): set the nudge cron's `deliver` to `telegram:the operator` (or the
user's chat) so every fire pings them. For the operator's lab the standing nudge cron is
`c9278f0dbe5e` (ULTRON Stay-Awake Nudge, `*/2 * * * *`) — keep its deliver on
`telegram:the operator`. For crons that should stay silent when healthy, `deliver: local`
is fine ONLY if something else (a watchdog that escalates on failure) surfaces
problems; a nudge whose whole job is "don't let me go silent" must be visible.

## Why these belong here
Both serve ULTRON Rules 2 (verify the ACTUAL deliverable — a blank dashboard is
a failed deliverable, not a working one) and 4 (never sleep — a silent nudge is
a slept nudge). A dashboard that looks dead and a nudge that vanishes are both
"looks fine, isn't" traps.
