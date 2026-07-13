# chrome.i18n can't switch languages at runtime — roll a dictionary layer

`chrome.i18n.getMessage` resolves against the *browser's* UI locale, fixed at browser startup, so an in-app language selector is impossible with the built-in API; a plain shared dictionary module plus a stored `language` setting is the workaround.

## Why it matters

"Add a language toggle to the extension" sounds like a `_locales/` job, but `chrome.i18n` gives users no per-extension choice — a Japanese speaker on an English-configured Chrome is stuck with English. If the requirement is a runtime toggle, skip `chrome.i18n` entirely rather than fighting it.

## How it works

- One shared `i18n.js` holding the dictionary (`{ en: {...}, ja: {...} }`) and lookup helpers `t(key, lang)` / `tFormat(key, lang, params)`. Load it in every context that renders text: `<script>` tag in the popup, manifest `content_scripts` entry (and any manual-injection file list) for content scripts.
- The chosen language lives in `chrome.storage.sync` as a normal setting.
- Static markup gets `data-i18n` / `data-i18n-placeholder` attributes; a `applyTranslations(lang)` pass swaps them all, so toggling re-renders without a page reload.
- JS-built strings go through `t()` against a module-level current-language variable.
- Messages from the background to content scripts **carry the language in the payload**, so the receiving script renders in the right language without doing its own storage round-trip.

## Gotchas

- User-controlled values interpolated into translated templates (hostnames, titles) must go in via `textContent` on a created element, never string-concatenated into `innerHTML` — split the template on the placeholder and build DOM nodes.
- Keep genuinely language-neutral content (numbers, math expressions) out of the dictionary; only translate UI chrome.
