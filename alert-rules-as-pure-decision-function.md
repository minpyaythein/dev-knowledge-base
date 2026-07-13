# Alert rules as a pure decision function — transitions + reminder caps

Put all notification logic in one pure function `decideAlerts(context) → Alert[]`, keyed on *state transitions* rather than states, so alerts fire once when something changes instead of on every check.

## Why it matters

Naive alerting ("if in stock → notify") spams a ping every check cycle while the condition holds. And alert logic scattered through the checker (a send here, an if there) is untestable — you can't unit-test "notify only on the third consecutive failure" if it's tangled with fetching and DB writes.

## How it works

The function takes a plain context object and returns a list of alerts to send — no I/O inside:

```ts
decideAlerts({ prevOk, curr, manual, targetPrice, targetCurrency,
               consecutiveFailures, stockNotifiedAt, priceNotifiedAt, now })
```

Three patterns cover most alerting needs:

- **Transition, not state.** In-stock pings when `prevOk.inStock === false && curr.inStock === true` (out→in), not whenever it's in stock. Price pings when the price *crosses* to/below the target, not while it sits there.
- **Reminder cap for persistent conditions.** While the condition holds, allow a repeat ping at most once per 24 h, tracked by a per-alert-kind `lastNotifiedAt` timestamp stamped after each successful send. `null` timestamp = never sent = due.
- **Exactly-N for failures.** "Page broke" warns once, at *exactly* 3 consecutive failures (`=== 3`, not `>= 3`) — a transient timeout stays silent, a persistent break alerts exactly once.

Asymmetries are deliberate and belong in the rules, not the caller: user-initiated manual checks always ping when in stock (the user asked "is it available *now*?"); price *rises* never ping (nobody wants that alert).

Because it's pure, every rule is a one-line unit test: build a context, assert the returned list.

## Example

Cron ticks while a product stays in stock for 3 days. Tick 1 after restock: transition → ping, stamp `stockNotifiedAt`. Next ~95 ticks: in stock but no transition and not 24 h yet → silent. Tick at +24 h: reminder due → one ping, re-stamp. Total: 4 pings in 3 days instead of ~290.
