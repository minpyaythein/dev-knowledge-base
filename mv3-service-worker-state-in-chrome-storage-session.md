# MV3 service worker state belongs in chrome.storage.session, not variables

A Manifest V3 service worker can be killed by Chrome at any idle moment, so runtime state kept in module-level variables silently vanishes; `chrome.storage.session` survives worker restarts within the same browser session.

## Why it matters

MV2 background pages were persistent, so `let elapsed = {}` at module scope just worked. In MV3 the worker is torn down after ~30s of inactivity and re-spun on the next event — every in-memory variable resets to its initial value. A timer extension that keeps its counters in memory loses all accumulated time on each teardown, and the bug is intermittent because it depends on when Chrome decides to kill the worker.

## How it works

Split state by lifetime:

| State | Where | Lifetime |
|---|---|---|
| User settings (tracked sites, thresholds) | `chrome.storage.sync` | Persistent, synced across devices |
| Runtime counters (elapsed seconds, active hostname, flags) | `chrome.storage.session` | Browser session — survives SW restarts, cleared on browser exit |
| Truly ephemeral handles (timeout IDs) | In-memory variable | One SW lifetime — must be tolerable to lose |

Every handler re-reads state from storage at the start instead of trusting a cached copy:

```js
async function getSessionState() {
    return chrome.storage.session.get({
        activeHostname: null,
        elapsed: {},
        lastActiveTime: null
    });
}
```

The defaults object passed to `get()` doubles as the schema — a fresh worker after restart gets sane values without an init step.

## Gotchas

- Anything kept in memory (e.g. a `pendingTimeouts` map of `setTimeout` IDs) must be designed so losing it is harmless — pair it with a recurring `chrome.alarms` backstop that re-derives the schedule from stored state.
- `chrome.storage.session` is cleared when the browser closes; if state must outlive that, use `chrome.storage.local` instead.

## See also

- `~/Desktop/Development/knowledge/promise-chain-mutex-for-chrome-storage-rmw.md`
- `~/Desktop/Development/knowledge/chrome-alarms-sub-30s-settimeout-dual-path.md`
