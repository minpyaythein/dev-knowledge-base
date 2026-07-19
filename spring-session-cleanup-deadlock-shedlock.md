# Spring Session cleanup deadlocks when the app runs on 2+ instances

Spring Session's default expired-row `DELETE` has no `ORDER BY`, so two app instances lock the same rows in opposite order and deadlock.

## Why it matters

Scaling a Spring Boot app to 2+ instances for high availability silently turns the built-in 60-second session cleanup into a recurring InnoDB deadlock. The redundancy meant to add safety is exactly what triggers the bug.

## How it works

Spring Session runs `DELETE FROM spring_session WHERE EXPIRY_TIME < ?` every 60s on every instance. With no `ORDER BY`, delete order is undefined, so two instances can each hold one row's exclusive lock while waiting for the other's.

| time | instance 1 | instance 2 | result |
|------|--------|--------|--------|
| 00 | lock row A ✓ | lock row B ✓ | each holds a different row |
| 01 | wait for B ✗ | wait for A ✗ | mutual wait → **deadlock** |

## Example

Two complementary fixes: order the delete (removes the deadlock) and elect a single runner via ShedLock (removes the wasted double-run).

```sql
-- ordered + bounded: fixed lock order, short transaction
DELETE s FROM SPRING_SESSION s
INNER JOIN (
    SELECT PRIMARY_ID FROM SPRING_SESSION
    WHERE EXPIRY_TIME < ? ORDER BY EXPIRY_TIME LIMIT 100
) t ON s.PRIMARY_ID = t.PRIMARY_ID;
```

ShedLock inserts one row into a PK'd `shedlock` table before cleanup; only the first instance wins the insert and runs, the rest skip.

## Gotchas

- `ORDER BY` alone stops the deadlock but not the wasted concurrent run; ShedLock stops the run itself.
- `LIMIT 100` keeps each transaction short so it holds few row locks; leftovers are cleaned on the next 60s tick.
- ShedLock's `lock_until` auto-releases if the holding instance crashes, so a new instance takes over next tick.

## See also

- [pdfbox-download-unbounded-heap-oom.md](pdfbox-download-unbounded-heap-oom.md) — another incident where an OOM crash is exactly the case ShedLock's `lock_until` recovers from.
