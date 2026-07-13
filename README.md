# dev-knowledge-base — one hard-won coding learning per file

<div align="center">

[![English](https://img.shields.io/badge/README-English-2563eb?style=for-the-badge)](README.md)
[![日本語](https://img.shields.io/badge/README-日本語-lightgrey?style=for-the-badge)](README.ja.md)

</div>

A personal knowledge base of coding insights extracted from real projects. Each markdown file captures exactly **one** non-obvious learning — a platform quirk, a race condition, a design pattern that earned its keep — written so it can be understood cold, months later, with zero project context.

---

## Why I keep this

Debugging a subtle race or discovering a platform limitation is expensive; re-discovering the same thing six months later in a different project is pure waste. So at the end of a working session I extract what just became clear into a standalone note, verified against the actual code before it's written down.

- **One topic per file** — a note answers one question well instead of ten questions vaguely, which makes it findable by filename alone.
- **Timeless over diary** — notes state what's true ("X works because Y"), not "today I learned", so they don't rot with the project they came from.
- **Verified, not remembered** — every claim points back to code that was read or behavior that was observed; fuzzy specifics get omitted rather than guessed.

---

## Note format

Every note follows the same skeleton, targeting under one screen:

```markdown
# Title — the insight itself, not just the topic

One-line TL;DR.

## Why it matters
## How it works
## Example        (optional)
## Gotchas        (optional)
## See also       (optional — links to sibling notes)
```

---

## The notes

### Chrome extensions (Manifest V3)

| Note | Insight |
|---|---|
| [MV3 service worker state](mv3-service-worker-state-in-chrome-storage-session.md) | Runtime state belongs in `chrome.storage.session`, not module variables — the worker dies at will. |
| [Sub-30s timers dual path](chrome-alarms-sub-30s-settimeout-dual-path.md) | `chrome.alarms` can't fire at short delays; pair `setTimeout` with a periodic-alarm backstop. |
| [Promise-chain mutex for storage](promise-chain-mutex-for-chrome-storage-rmw.md) | Serialize concurrent read-modify-writes of `chrome.storage` with a 5-line promise chain. |
| [Single writer for runtime state](extension-background-single-writer-runtime-state.md) | Only the background mutates runtime state; UI contexts send messages instead of writing. |
| [Inject-then-retry messaging](content-script-sendmessage-inject-fallback.md) | `tabs.sendMessage` fails on pages loaded before install — inject the script and resend. |
| [Orphaned content scripts](orphaned-content-script-safe-sendmessage.md) | Extension reloads orphan running content scripts; wrap `chrome.runtime` calls in try/catch. |
| [Runtime language toggle](chrome-i18n-runtime-language-toggle.md) | `chrome.i18n` is fixed to the browser locale — a runtime toggle needs your own dictionary layer. |

### Streamlit

| Note | Insight |
|---|---|
| [`st.cache_resource` as server state](st-cache-resource-as-process-global-server-state.md) | The resource cache doubles as process-global server-side state. |
| [Iframe sandbox escape (legit)](streamlit-iframe-sandbox-parent-realm-injection.md) | `components.html` runs in an iframe that can't navigate the page — inject into the parent realm. |
| [Rerun vs reload identity](streamlit-rerun-vs-reload-client-identity.md) | Carry client identity in `st.query_params`, set from Python, to survive both reruns and reloads. |

### Cloudflare & serverless

| Note | Insight |
|---|---|
| [Cron floor + app gating](cloudflare-cron-floor-plus-app-gating.md) | User-configurable schedules on a fixed cron: fire at the floor frequency, gate in code. |
| [One Worker, full stack](single-worker-api-plus-static-spa.md) | A single Cloudflare Worker serving API + SPA + cron is the $0 full-stack shape. |

### Ops & reliability

| Note | Insight |
|---|---|
| [Liveness vs app health](healthz-edge-liveness-vs-in-app-error-monitoring.md) | A platform health endpoint proves the host is up, not that the app works — monitor both layers. |
| [Discord webhook 403](discord-webhook-403-python-urllib-user-agent.md) | Discord blocks Python's default urllib User-Agent — set your own. |

### Design & data patterns

| Note | Insight |
|---|---|
| [Alert rules as pure function](alert-rules-as-pure-decision-function.md) | Model alerting as a pure decision function over transitions, with reminder caps. |
| [Append-only snapshots](append-only-snapshots-as-source-of-truth.md) | Append-only snapshots as the single source of truth; everything else is a view. |
| [Money as integer minor units](money-as-integer-minor-units.md) | Store money as integer minor units plus a separate ISO 4217 currency column. |
| [Extraction cascade](generic-product-extraction-cascade.md) | Extract product data by cascading from structured sources down to guessed ones. |
| [Flush timers by whole units](timer-flush-whole-seconds-carry-remainder.md) | Advance the flush cursor by units counted, not to `now()` — or the timer drifts slow. |

### Frontend

| Note | Insight |
|---|---|
| [Dark mode without the flash](dark-mode-without-flash-inline-script.md) | Apply the saved theme in an inline `<head>` script, before first paint. |

### LLM

| Note | Insight |
|---|---|
| [Smoothing bursty token streams](smoothing-bursty-llm-streams-reader-thread-pacer.md) | Smoothing a bursty LLM stream needs a reader thread + pacer, not a pull-based throttle. |

---

## Adding a note

Notes are captured with a custom Claude Code `/learnings` skill at the end of a working session: pick the one thing that just became clear, verify every claim against the code, write it into a narrowly-named kebab-case file. Rules of the house: one topic per file, timeless phrasing, honest about uncertainty, under ~50 lines.
