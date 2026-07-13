# One writer for extension runtime state: UI sends messages, background mutates

In a multi-context extension (background + popup + content scripts), exactly one context — the background — should write runtime state; every other context requests mutations via messages, because cross-context writes bypass any locking and race the owner's in-flight bookkeeping.

## Why it matters

The popup *can* write `chrome.storage.session` directly, and it's tempting ("just delete the counter key here"). But the background may hold un-flushed derived state — e.g. seconds accrued since the last write that only it knows about. A popup write lands mid-computation: the background's next flush re-adds time to a counter the popup just zeroed, or overwrites the popup's change wholesale. The background's internal mutex can't help; the popup never goes through it.

## How it works

- **Popup/content scripts read freely, write never.** Reading session state for display is safe; mutations go out as typed messages (`RESET_SITE`, `THRESHOLD_CHANGED`, `DISMISS_POPUP`).
- **The background handler does the ordering-sensitive prep** the caller can't: flush pending derived state *first*, then apply the mutation, all inside its serialization lock.

```js
// background — RESET_SITE handler
runExclusive(async () => {
    const state = await flushElapsed(await getSessionState()); // bank active time FIRST
    delete state.elapsed[host];
    // only restart the shared clock if the reset site IS the active one
    if (state.activeHostname === host) patch.lastActiveTime = Date.now();
    await setSessionState(patch);
});
```

The flush-first step is why this must live in the background: resetting site B while site A is active must not disturb A's running timer, and only the background knows A's un-banked seconds.

- **Persistent *settings* are different** — the popup writing `chrome.storage.sync` directly is fine, because the background treats settings as read-only input, not state it's mid-mutation on.

## See also

- `~/Desktop/Development/knowledge/promise-chain-mutex-for-chrome-storage-rmw.md`
