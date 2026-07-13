# tabs.sendMessage fails on pre-install pages — inject, then retry

Pages already open when an extension is installed or reloaded have no content script, so `chrome.tabs.sendMessage` errors; the fix is to catch that, inject via `chrome.scripting.executeScript`, and resend — after checking the URL is scriptable at all.

## Why it works this way

Manifest `content_scripts` only inject on page load. Any tab loaded before install/update is a dead zone until it's reloaded — a background message to it fails with "Could not establish connection". Users never reload all their tabs, so without a fallback the extension silently doesn't work on exactly the tabs the user has been sitting on.

## How it works

```js
chrome.tabs.sendMessage(tabId, payload, () => {
    if (!chrome.runtime.lastError) return; // delivered

    if (!canInjectInto(tab.url)) return;   // chrome://, Web Store, file:// — skip

    chrome.scripting.executeScript({ target: { tabId }, files: ['i18n.js', 'content.js'] })
        .then(() => chrome.scripting.insertCSS({ target: { tabId }, files: ['content.css'] }))
        .then(() => setTimeout(() => chrome.tabs.sendMessage(tabId, payload, () => {
            if (chrome.runtime.lastError) { /* log softly, give up */ }
        }), 100))
        .catch(() => { /* error pages etc. still refuse injection — expected */ });
});
```

Key pieces:

- **Gate on URL first**: `/^https?:\/\//` — injecting into `chrome://`, the Web Store, or error pages throws. Filter cheaply instead of catching noisily.
- **Inject dependencies in order**: if the content script relies on other files (a shared i18n dictionary), list them before it in `files`.
- **Small delay before retry** so the injected script's listener is registered.
- **Idempotency guard in the content script**: manual injection can land on a page that already has the script, so guard listener registration (`if (!window.__myListener) { window.__myListener = true; addListener(...) }`) or every message is handled twice.

## Gotchas

- Some `https://` pages still refuse injection (Chrome error pages, Web Store). Treat the `.catch` as expected, not actionable — log at warn level and move on.
