# Streamlit `components.html` iframe can't navigate the page — inject scripts into the parent realm

A script inside Streamlit's `components.html` iframe cannot redirect the top page (the iframe is sandboxed without top-navigation), but it *can* reach the parent DOM (`allow-same-origin`) — so append your `<script>` to the parent document and let it run there, un-sandboxed.

## Why it matters

Third-party widgets like Cloudflare Turnstile need a success callback that navigates the page (e.g. to carry a token back in the URL). Hosting the widget inside `components.html` silently fails: the browser attributes the navigation to the sandboxed context and blocks it, with no error surfaced to the user. The trap also bites plain `location.replace` calls used for cookie/URL reconciliation.

## How it works

- `st.markdown` strips `<script>` tags, so raw script can't go in the main page directly.
- `components.html` renders in an iframe sandboxed **without** `allow-top-navigation` but **with** `allow-same-origin`.
- The escape hatch: use a 0-height `components.html` iframe purely as an **injector**. Its script grabs `window.parent.document`, creates a `<script>` element with the real logic as text, and appends it to the parent's `<head>`. That code now executes in the parent's realm, where navigation is allowed.
- Give injected elements stable ids (`cf-cb`, `cf-api`) and check for them first — Streamlit reruns re-execute the injector, and the injection must be idempotent.

## Example

Turnstile gate in a Streamlit app: `st.markdown` draws an empty `<div id="cf-slot">` in the main page; a 0-height injector iframe appends (1) a parent-realm callback that sets `?cf_token=...` and navigates, and (2) the Turnstile api.js tag, to the parent document. The widget mounts into `#cf-slot`; on success the parent-realm callback navigates freely and Python reads the token back via `st.query_params`.

## Gotchas

- The injector iframe and the `st.markdown` mount point are separate Streamlit elements with **no render-order guarantee** — on a cold load the script can run before the slot exists. Poll for the slot instead of bailing (`if (!slot) return;` leaves a dead gate).
- Scroll events don't bubble: to watch scrolling on Streamlit's inner container from injected code, use a capture-phase listener on `window.parent`.
