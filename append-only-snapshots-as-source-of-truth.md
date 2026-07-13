# Append-only snapshots as the single source of truth

For anything you observe repeatedly over time (prices, stock, statuses), store each observation as an immutable row and *derive* everything else — never keep a mutable "current state" column.

## Why it matters

The obvious design — a `current_price` column updated on every check, plus maybe a history table — creates two copies of the truth that drift apart, and every new feature ("when did it last change?", "alert on transitions") needs another mutable column and another update path. One append-only table answers all of them for free.

## How it works

Every check appends one **snapshot** row: timestamp, status (ok/failed), and the observed fields (price, currency, in-stock — nullable, where `null` means "unknown", never a guessed default). Nothing is ever updated in place. Then:

| Question | Answer |
|---|---|
| Current state? | the latest snapshot |
| History / chart? | all snapshots, ordered by time |
| Did it just change? (alerting) | compare the two newest snapshots |
| How long broken? | count consecutive non-ok snapshots from the end |

The discipline: when a feature tempts you to add a mutable status column, derive it from snapshots instead. The only mutable state that's legitimate is *bookkeeping about actions you took* (e.g., "last time an alert was sent"), which isn't an observation.

## Gotchas

- **Read the comparison baseline before inserting the new snapshot.** "Previous snapshot" queried after the insert returns the row you just wrote, and every transition looks like no-change. Order: read latest → insert new → compare.
- Unbounded growth is real but cheap to handle later (prune rows older than N days); don't let it push you back to mutable state.

## Example

A product goes out of stock, then restocks. No code ever flips an `in_stock` flag — the checker just appends `in_stock=0`, later `in_stock=1`. The dashboard shows "In stock" (latest row), the timeline shows the gap (all rows), and the alert logic sees `prev=0, curr=1` → out→in transition → send the ping.
