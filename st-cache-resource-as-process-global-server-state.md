# `@st.cache_resource` doubles as process-global server-side state in Streamlit

`@st.cache_resource` returns the same object to every session and rerun, so a zero-arg cached function returning a dict gives you a process-wide mutable store — the Streamlit equivalent of a Node module-level `const buckets = new Map()`.

## Why it matters

Streamlit's obvious state home, `st.session_state`, dies on page reload (a reload = new session). Anything that must survive a reload — rate-limit buckets, abuse counters, cross-session caches — needs to live server-side, and Streamlit has no explicit "app state" API. `cache_resource` fills that gap without a database.

## How it works

```python
@st.cache_resource
def _rate_state() -> dict:
    return {"buckets": {}, "last_prune": 0.0}
```

Every caller in every session gets the *same* dict; mutations are visible everywhere. A browser reload can't reset it — only a process restart does (e.g. the host's container waking from sleep), which is usually an acceptable bound for a cost guardrail.

The same decorator with an argument is the standard expensive-work cache: `@st.cache_resource` on `build_retriever(file_bytes)` keyed on the PDF's bytes means Streamlit's rerun-everything-per-interaction model doesn't re-embed the document on every question.

## Example

A fixed-window rate limiter: buckets of `{count, reset_at}` in the shared dict, keyed per client. `consume_limit` starts/increments/rejects against the window; a read-only `peek_limit` feeds a usage meter without spending a slot. Reloading the page doesn't reset the count, because the state never lived in the session.

## Gotchas

- It's in-memory and per-process: state resets on restart, and it won't be shared across replicas if the host scales out. Upgrade path is an external store (e.g. Redis).
- Prune expired entries opportunistically — the process can live a long time and the dict grows forever otherwise.
