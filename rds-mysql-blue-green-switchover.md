# RDS MySQL blue/green switchover runbook

Drain the app to zero DB connections before triggering switchover, then verify green is writable before reopening traffic; rollback is a slow point-in-time restore.

## Why it matters

The writer endpoint name stays the same, but every open connection is killed at the swap, and old-blue (the pre-switchover instance) stops syncing afterward. Every minute of writes on green widens the data a rollback would lose, so validate fast.

## Steps

1. Disable any scheduled jobs (e.g. EventBridge rules) so nothing fires mid-switchover.
2. Put up the maintenance page (503) to block new user traffic.
3. Wait 30–60s for in-flight requests to finish, then stop the backend app instances (this closes the connection pools cleanly).
4. Confirm no active DB sessions — `Sleep` rows are fine, switchover kills them anyway:
   ```bash
   mysql -h <writer-endpoint> -u <admin> -p -e "
     SELECT id, user, host, command, time FROM information_schema.processlist
     WHERE command != 'Sleep' AND user NOT IN ('rdsadmin','rds_monitoring');"
   ```
5. Trigger the switchover in the RDS console and wait for the new instance to show **Available** (both instances may briefly report read-only during the swap).
6. Verify DNS now points at green:
   ```bash
   dig +short <writer-endpoint>
   ```
7. Verify version and writability — expect the target version and `0`:
   ```bash
   mysql -h <writer-endpoint> -u <admin> -p -e "SELECT VERSION(); SELECT @@read_only;"
   ```
8. Do a round-trip write:
   ```bash
   mysql -h <writer-endpoint> -u <admin> -p <<'SQL'
   CREATE TABLE IF NOT EXISTS _switchover_check (id INT, ts DATETIME);
   INSERT INTO _switchover_check VALUES (1, NOW());
   SELECT * FROM _switchover_check;
   DROP TABLE _switchover_check;
   SQL
   ```
9. Start the backend and confirm health shows `db: UP`:
   ```bash
   curl -s https://<backend-host>/actuator/health | jq .
   ```
10. Remove the maintenance page and re-enable the scheduled jobs from step 1.
11. Watch the slow query log, app logs, and metrics for 15–30 minutes — a new MySQL version may pick different query plans.

## Gotchas

- Old-blue sits at `<endpoint>-old1` but is not kept in sync; rollback means point-in-time restore, not a re-swap.
- The first connection per app user after switchover hits a cold `caching_sha2_password` cache, so expect an auth latency spike at restart.
- InnoDB defaults change across major versions (e.g. 8.4: `innodb_io_capacity` 200→10000, adaptive hash index OFF) — that's what the monitoring window in step 11 is for.

## See also

- [mysql-caching-sha2-password-connection-flow.md](mysql-caching-sha2-password-connection-flow.md) — why the first post-switchover connection is slow.
