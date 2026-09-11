# Process restart + served-state verification

Rules for local servers/panels you (re)launch, and for verifying what a browser pane actually shows. Generalizes to any dev server the agent starts and then edits.

## Restart discipline (never stack servers)
- Never restart a local server as `pkill -f <name>; start` in ONE command. The pkill pattern matches the invoking wrapper shell itself, so the shell dies (exit -9) before the start fires — while an OLDER instance keeps the port. Stale servers then stack, each serving a different HTML build.
- Kill by explicit PID list with the bracket trick: `for p in $(pgrep -f '[n]ame.py'); do kill -9 $p; done`. The bracket stops the pattern from matching the shell that runs it.
- After killing, confirm the port is actually free: `ss -tlnp | grep <port>` must be EMPTY. Then start exactly ONE instance (`terminal(background=true)`; no notify for never-exit servers).
- Prove the SERVED artifact is the new build: `curl -s <url> | grep -c <build-marker>`. The listening process, not your intent, decides what the user sees.
- Process-count checks: `pgrep -f <name> | wc -l` over-counts by one when the launcher wrapper's cmdline contains the literal name — count via `ss -tlnp` listeners on the port, or strip the wrapper.

## stdlib http.server serves the HTML captured at process START
- Any HTML/JS edit requires a server restart — there is no hot reload. If page behavior looks stale, suspect an old process before suspecting the browser.
- Send `Cache-Control: no-store` on every response while iterating — webviews/preview panes will otherwise resurrect a previous build from cache after the server is fixed.

## Verify page interactivity IN-PAGE, not through click proxies
- Synthetic click reports from browser-driving tools can claim success ("acted: clicked ...") while the page's JS never fired. Ground truth = an in-page probe: build a `?demo=1` query-param mode that programmatically `.click()`s the target and renders VISIBLE state, then read the page text/state back.
- Add a tiny version probe to the page (e.g. a title badge carrying build markers) so any load can be fingerprinted fresh-vs-cached at a glance.
- A byte-identical page read before vs after an edit proves nothing when the added markup renders empty — put observable state (counts, markers) in the probe, not silent divs.

## Debugging order for "clicks do nothing" on a served panel
1. `curl -s <url> | grep -c <new-build-marker>` — 0 = an old process still owns the port → restart discipline above.
2. Open `<url>?demo=1` — the in-page auto-click proves the whole handler chain independent of who clicks.
3. If demo works but tool clicks don't: the click-delivery layer is the problem, not the page. Tell the user to test with a real mouse; do not keep re-synthesizing clicks.
4. The user's real mouse is the final test; report the page verified only after the served-marker + demo checks pass.
