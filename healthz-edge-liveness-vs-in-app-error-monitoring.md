# A platform health endpoint proves the host is up, not that the app works — monitor both layers

An uptime monitor on a health URL answered at the hosting platform's edge keeps saying `ok` while the app's actual functionality (API calls, RAG pipeline) is failing — so pair it with an in-app error mirror that reports genuine upstream failures from inside the request path.

## Why it matters

It's tempting to point UptimeRobot at a health endpoint and call monitoring done. But on Streamlit Community Cloud, `/healthz` is answered at the edge layer: it proves the container/platform is alive, nothing more. A broken API key, an exhausted quota, or a dead upstream provider leaves the monitor green while every user request errors.

## How it works

Two layers, each covering the other's blind spot:

| Layer | Watches | Catches | Misses |
|---|---|---|---|
| 1. External keyword monitor on `/healthz` | Platform liveness from outside | Host down, app asleep, DNS/TLS breakage | App-level failures behind a healthy edge |
| 2. In-app error mirror (Discord webhook) | Real exceptions in the request path | Upstream API failures, bad keys, rate limits | Nothing arriving at all (host down = no code runs) |

Layer 2 placement rules that made it clean:
- Hook the **existing `except` seams** around real upstream calls — don't add a top-level catch-all (frameworks like Streamlit use control-flow exceptions such as `RerunException` constantly; a global catch is noisy and fragile).
- Alert only on the 5xx analog: user-input failures (e.g. a scanned PDF with no text) are the 400 analog and stay silent.
- Fire-and-forget on a daemon thread, fully swallowed, throttled per error signature (route + type + message) so a looping failure can't machine-gun the channel — and the throttle resets on restart, so you learn the error is *still* happening after a deploy.

## Gotchas

- Streamlit Cloud's `/_stcore/health` 303-redirects anonymous clients into an auth/wake dance even when the app is awake, so a plain keyword monitor never sees `ok` there — use `/healthz`, which answers `{"status":"ok"}` directly.
- The Layer-2 webhook must gate on its env var being unset (= off) so local dev doesn't need the secret.

## See also

- `~/Desktop/Development/knowledge/discord-webhook-403-python-urllib-user-agent.md` — the silent failure mode that almost sank Layer 2.
