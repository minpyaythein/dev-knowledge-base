# MySQL `caching_sha2_password`: fast path vs slow path

A connection needs the slow RSA path (and Python's `cryptography` package) only when the server's in-memory auth cache is empty for that user.

## Why it matters

PyMySQL needs `cryptography` only on the cache-miss path, so it works in dev (warm cache) and crashes in production after a server restart, failover, or blue/green swap. If your code survives a fresh server's first connection, it survives everything.

## How it works

The server keeps an in-memory map of user → password hash. It is cleared by a `mysqld` restart, not by `FLUSH PRIVILEGES`, and is shared by all clients (Java, Python, CLI). Each new connection sends a hash-based scramble; the server's reply decides the path:

| Cache | Server reply | Client needs |
|---|---|---|
| hit | `0x03` fast auth success | `hashlib` only (always available) |
| miss | `0x04` perform full authentication | RSA-encrypt password → `cryptography`, or TLS (sends plaintext over the encrypted channel instead) |

A successful slow-path auth populates the cache, so every later connection from any client takes the fast path.

## Example

```
1st connection after restart:  scramble → 0x04 → fetch RSA key → encrypt → OK  (cache filled)
2nd connection onward:         scramble → 0x03 fast_auth_success → OK
```

## Gotchas

- A Multi-AZ failover lands on a different server with its own empty cache — the first connection is slow-path again.
- The plugin is the default since MySQL 8.0.4; code that "worked on 8.0 without `cryptography`" was riding a warm cache.

## See also

- [rds-mysql-blue-green-switchover.md](rds-mysql-blue-green-switchover.md) — the switchover that empties this cache.
