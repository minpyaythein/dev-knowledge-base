# Streamlit: rerun vs reload — carry client identity in `st.query_params`, set from Python

On Streamlit, a **rerun** re-executes the script over the live websocket (session kept), while a **page reload** opens a fresh session (`st.session_state` lost, session-gated features re-fire). So any client id or gate state that must survive a URL change has to be written from **Python (rerun)**, never via **JS (reload)**.

## Why it matters

A public Streamlit app needs a stable per-client key for rate limiting, and the obvious identifiers all fail on Streamlit Community Cloud. Getting this wrong makes limits reset on reload, leak across users, or force visitors through a bot gate twice.

## How it works

What fails on Streamlit Community Cloud (verified live):

| Candidate | Why it fails |
|---|---|
| `X-Forwarded-For` | Carries only Streamlit's *internal* LB hops (rotating private 10.x IPs) — all users collapse onto ~2 buckets that flip per request |
| JS-set cookie via `st.context.cookies` | Cookie visible in DevTools, but reads back **empty** server-side |
| Cookie + JS reload to copy it into the URL | The reload starts a new session, so a session-based bot gate re-challenges — a jarring double-gate |

What works: `st.query_params["cid"] = uuid4().hex` from Python. Setting a query param triggers a rerun — not a reload — so the session (and a passed Turnstile gate) survives, and the id rides in the URL through reloads and duplicated tabs.

## Example

`client_id()` reads `st.query_params.get("cid")`; if absent it mints a uuid and writes it back. The Turnstile success callback rebuilds the URL from `location.href`, so `cid` is preserved through the gate while only the one-time `cf_token` is stripped afterwards. Rate buckets live server-side (see the process-global store pattern), keyed by `cid`.

## Gotchas

- A fresh bare-URL visit (or incognito) mints a new id — acceptable for a cost guardrail, not for security.
- Ordering matters in debugging this: an early dead branch (the XFF check) can short-circuit a correct fallback below it, hiding that the fallback never ran.
