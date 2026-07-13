# Discord webhooks 403-block Python's default urllib User-Agent

POSTing to a Discord webhook with stdlib `urllib` fails with HTTP 403 unless you set an explicit `User-Agent` header — Discord's Cloudflare layer blocks the default `Python-urllib/x.y` UA.

## Why it matters

The failure mode is invisible by design: alerting code deliberately swallows its own errors (an alert failure must never escalate out of an error handler), so the 403 leaves no trace except missing alerts. You only notice when an incident happens and no Discord message ever arrived. Node's `fetch` sends a browser-ish UA and never hits this, so a port from a working Node implementation breaks silently.

## How it works

```python
request = urllib.request.Request(
    webhook_url,
    data=json.dumps(payload).encode("utf-8"),
    headers={
        "Content-Type": "application/json",
        "User-Agent": "MyApp-AlertBot/1.0 (+https://myapp.example.com)",
    },
    method="POST",
)
urllib.request.urlopen(request, timeout=5)
```

Verified directly: the same payload returns 403 without the UA header and 204 with it. Any descriptive non-default UA works.

## Gotchas

- When porting webhook code between runtimes, check what UA the original runtime sent — the header you never wrote may be the thing that made it work.
- While debugging "swallowed" integrations, temporarily log the caught exception (`print` in the except) — the 403 was sitting right there.
