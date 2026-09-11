# Pipeline engagement verification — 'component alive' is not 'pipeline armed'

## The failure pattern
Multi-component pipelines (interceptor→sink, driver→collector,
producer→consumer, watcher→actor) fail HALF-ALIVE: one component runs green
while the other never deployed. Liveness checks that match process names,
port binds, or 'a log file exists' report SUCCESS while the pipeline delivers
zero work. The user then spends real time (a 37-minute play session, an
overnight batch) feeding a corpse.

## Rule
Before declaring a pipeline live, verify the SUCCESS LINE of the ENGAGING
component — the one whose job is to start moving the work:
- capture tap: the interceptor's first INTERCEPTED item, or the sink
  transitioning PAST its 'Waiting for connection' state once the workload
  started. A sink still saying 'Waiting' = the engage half is DEAD.
- watcher/cron: first real triggered action, not 'watcher running'.
- build+deploy: the deploy target's confirmation, not build exit 0.
- form/webhook: receiving-side delivery evidence, never the HTTP 200.

## Session instance (2026-09-02, Aeternum live capture)
proxy_capture.py = udp_redirect (WinDivert intercept) + DTLS proxy (sink).
A CommandLine match on 'proxy_capture' reported 'TAP ALIVE' while the
redirect half had silently died at driver open; game traffic flowed direct
to the server; the sink logged 'Waiting for game client connection' for the
entire 37-minute session. Correct check: sink must transition past
'Waiting' within seconds of the workload starting; if not, go read the
ENGAGE half's output, not the sink's.

## Operational addendum
- First runs of a pipeline: visible or captured output per component.
  Hidden windows swallow the exact error that explains a zero. Redirect
  stdout/stderr to per-component files unconditionally.
- Half-dead discovery = tell the user immediately and plainly ('armed but
  never engaged; your session was not captured'). Never let them keep
  feeding it.
