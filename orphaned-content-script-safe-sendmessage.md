# Orphaned content scripts throw "Extension context invalidated"

Reloading or updating an extension orphans every content script already running in open pages — their `chrome.runtime` calls then throw synchronously; wrap sends in try/catch and go quiet, because nothing can be done until the page reloads.

## Why it matters

During development you reload the extension constantly, and every open page with the old content script becomes a landmine: the next user interaction that triggers `chrome.runtime.sendMessage` throws `Extension context invalidated` into the page console. It looks like a real bug and can break the script's remaining UI (an overlay that won't dismiss), but it's an unavoidable lifecycle state, not an error to fix.

## How it works

The orphaned script keeps running in the page, but its bridge to the extension is severed — the old extension context is gone and the new one doesn't know this script exists. Recovery is impossible from inside the script; only a page reload gets a fresh, connected copy. So the correct handling is to swallow:

```js
function safeSendMessage(message) {
    try {
        chrome.runtime.sendMessage(message);
    } catch (e) {
        // orphaned content script; ignore — page reload brings a fresh one
    }
}
```

Route every `chrome.runtime` call in the content script through this wrapper. Degrade gracefully: if the script's UI action normally waits for the background's response, make the local effect (e.g. removing the overlay) happen regardless, so the orphaned script at least doesn't trap the user.

## Gotchas

- The throw is **synchronous**, unlike most extension messaging errors which arrive via `chrome.runtime.lastError` in a callback — so a `.catch()` on a promise chain won't necessarily catch it; you need `try/catch` around the call itself.
- The complementary problem (new extension can't message old pages) is solved separately by inject-and-retry from the background.

## See also

- `~/Desktop/Development/knowledge/content-script-sendmessage-inject-fallback.md`
