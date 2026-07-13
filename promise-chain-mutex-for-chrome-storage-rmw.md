# A promise-chain mutex serializes chrome.storage read-modify-writes

`chrome.storage` get→modify→set is not atomic and extension event handlers fire concurrently, so interleaved handlers clobber each other's writes; chaining every handler onto one promise makes them run one at a time.

## Why it matters

An alarm tick and a tab-switch handler can both read state, compute updates, and write back — the second write silently overwrites the first (dropped counter increments, flags that never stick). These races are invisible in casual testing because they need two events to land mid-flight of each other. A mutex is ~5 lines and removes the whole class.

## How it works

```js
let stateLock = Promise.resolve();

function runExclusive(fn) {
    const run = stateLock.then(fn, fn);
    stateLock = run.then(() => {}, () => {}); // keep the chain alive past errors
    return run;
}
```

Every entry point that touches shared state wraps its body:

```js
chrome.tabs.onActivated.addListener(() => runExclusive(async () => {
    await updateActiveTab();
    await checkThreshold('tab-activated');
}));
```

Two details carry the design:

- **The chain must survive rejections.** `stateLock = run.then(() => {}, () => {})` swallows errors *for the chain only* — the caller still gets the original rejection from `run`. Without this, one thrown error would permanently block every later handler.
- **Only entry points lock; internal helpers never do.** A helper called from inside a locked handler that calls `runExclusive` itself queues behind the very handler awaiting it — classic self-deadlock. The rule: handlers/listeners acquire, shared helpers assume they're already inside.

## Gotchas

- This serializes within one JS context (e.g. the service worker). Another context (popup page) writing the same keys bypasses the lock entirely — route those writes through the locking context via messages instead.

## See also

- `~/Desktop/Development/knowledge/extension-background-single-writer-runtime-state.md`
