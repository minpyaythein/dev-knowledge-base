# Flush timers by whole units counted, not to now() — or they drift slow

When banking elapsed time in integer seconds, advance the "last flushed" timestamp by exactly the seconds you credited (`last + delta*1000`), not to `Date.now()` — otherwise every flush discards the sub-second remainder and the timer systematically undercounts.

## Why it matters

An event-driven timer (flush on tab switch, focus change, periodic tick) flushes many times a minute. If each flush computes `delta = floor((now - last) / 1000)` and then sets `last = now`, up to 999ms vanish per flush. With frequent events the accumulated timer runs visibly slower than wall-clock — a drift bug that looks like "the timer is just inaccurate" and resists spot-checking because each individual loss is under a second.

## How it works

```js
const delta = Math.floor((now - lastActiveTime) / 1000);
if (delta <= 0) return; // < 1s accrued: flush nothing, keep lastActiveTime as-is

elapsed[key] += delta;
lastActiveTime += delta * 1000; // NOT `now` — remainder stays in the bank
```

The truncated fraction remains represented as "time since `lastActiveTime`", so the next flush picks it up. Over any interval the credited total equals wall-clock time floored once, instead of floored once per flush.

The same principle applies anywhere you convert a continuous quantity into integer units incrementally: consume exactly what you credited, carry the remainder.

## Gotchas

- The `delta <= 0` early-return must **not** touch `lastActiveTime` either — resetting it to `now` on a sub-second flush is the same bug through the side door.
- Only reset `lastActiveTime = Date.now()` at genuine restart points (counter reset, tracking resumed), where dropping the remainder is intended.
