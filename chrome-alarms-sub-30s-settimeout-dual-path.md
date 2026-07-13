# Sub-30-second timers in MV3 need a setTimeout + alarm dual path

`chrome.alarms` can't fire reliably at short delays (Chrome enforces a minimum granularity), so precise near-term wakeups need an in-memory `setTimeout` — with a periodic alarm as the backstop for when the service worker dies and takes the timeout with it.

## Why it matters

An extension that must act "in 12 seconds" can't use `chrome.alarms` — short delays get clamped or fire late. But a bare `setTimeout` in an MV3 service worker is also unreliable: if Chrome kills the worker before it fires, the timeout is gone. Neither mechanism alone works; the fix is to route by remaining time and accept bounded imprecision.

## How it works

When a deadline is computed, pick the mechanism by distance:

```js
const remaining = threshold - elapsed;
if (remaining < 30) {
    clearTimeout(pendingTimeouts[key]);
    pendingTimeouts[key] = setTimeout(() => check(), remaining * 1000);
} else {
    chrome.alarms.create('trigger-' + key, { delayInMinutes: remaining / 60 });
}
```

Three layers cover each other:

1. **One-shot alarm** for deadlines ≥30s away — survives SW restarts.
2. **`setTimeout`** for deadlines <30s away — precise, but lost if the SW dies.
3. **Recurring 30s periodic alarm** that re-runs the same check function — if layer 2's timeout was lost, the deadline fires at most one tick late.

The check function must be idempotent and re-derive everything from stored state (what's elapsed, what's the threshold, has it already fired), so it doesn't matter which of the three paths invokes it or how many times.

## Gotchas

- Clear both the timeout **and** the named alarm when the deadline becomes stale (target changed, entity removed) — otherwise the orphaned schedule fires a spurious wakeup later.
- `setTimeout` IDs live in a plain in-memory map; after an SW restart the map is empty, which is fine only because the periodic alarm backstops it.

## See also

- `~/Desktop/Development/knowledge/mv3-service-worker-state-in-chrome-storage-session.md`
