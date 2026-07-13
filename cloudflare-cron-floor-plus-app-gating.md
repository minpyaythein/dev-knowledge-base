# User-configurable schedules on a fixed cron: fire at the floor, gate in code

When the platform's cron is static config (Cloudflare Workers `wrangler.toml`) but users want a configurable schedule, fire the cron at the finest allowed granularity and let application code decide per tick whether to actually run.

## Why it matters

Workers cron triggers live in deploy-time config — you can't change them from a settings page without redeploying. Trying to sync user settings into platform config (via API calls at save time) is fragile and racy. Gating in code makes the schedule just a DB row.

## How it works

- `wrangler.toml` fires `*/15 * * * *` — the finest interval users may pick (15 min is also a sane free-tier floor).
- The user's schedule (interval, active-hours window, weekdays) is a JSON row in the DB, edited via a normal settings API.
- The `scheduled()` handler loads the schedule and asks a pure function `shouldRunNow(schedule, now)`; a "no" tick simply exits (idle ticks are effectively free).

`shouldRunNow` checks three things in the user's timezone:

- **Weekday** is in the allowed set.
- **Time window**: `start <= t < end`; if `start > end` the window wraps past midnight (e.g., 22:00–06:00). Support `"24:00"` as an end-of-day sentinel so "all day" is expressible as 00:00–24:00.
- **Interval alignment**: `minutesSinceLocalMidnight % intervalMinutes === 0`, so "every hour" means *on* the hour, deterministically — not "60 min since whenever the worker last ran".

Constrain the interval to multiples of the cron floor (15–1440, step 15); anything else can never align with real ticks.

## Gotchas

- The handler receives `controller.cron` (which spec fired). Use it to distinguish real ticks from manual triggers: `wrangler dev` never auto-fires cron, so local testing uses `curl "…/__scheduled?cron=manual"` — any spec other than the production one bypasses the gate on purpose, making manual triggers always run.
- Gate on `controller.scheduledTime`, not `new Date()` — the scheduled time is the aligned one; actual execution may start seconds later and fail the `% interval` check.
- For a fixed-offset zone like JST (UTC+9, no DST), timezone math is just `new Date(utc + 9h)` + `getUTC*()` reads. DST zones need a real timezone library.
