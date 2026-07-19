# A session JWT header verified per controller, with no security filter

When an API has no auth filter, each controller must call a verify step itself to check a session-token header on every request.

## Why it matters

Because auth is per-controller, a new endpoint that forgets to call the verify step is silently unauthenticated. If the token lives only in an in-memory SPA variable, a page reload drops it and forces a re-verify from the entry link.

## How it works

The client opens an entry link, and the server exchanges the link token for a session token it returns in a response header. The SPA keeps that token in memory and re-attaches it to every later call; each controller verifies it before doing work.

1. A verify endpoint checks the link token, then returns a fresh session token in a custom response header.
2. The SPA reads that header into an in-memory variable (not `localStorage`).
3. Before each call, the SPA sets the same custom header from that variable.
4. Each controller calls the verify service first.
5. That verifies RS256 with the session public key, requires `exp` and `iat`, looks up the authorization record, and requires its active flag set and reissue flag clear.

## Example

The token is captured once from the verify response, then replayed on every request; each protected endpoint guards itself.

```js
// SPA: capture from verify response, resend on every call
session.value = response.headers["x-session-token"]
http.defaults.headers.common["x-session-token"] = session.value
```
```java
// Server: first line of every protected controller method
AuthRecord a = verifyService.verifySession(request);
```

## Gotchas

- A separate admin app may use server-side sessions instead — don't assume both clients share this mechanism.
- The token is an in-memory variable, not `localStorage`, so reload or reopen loses the session and re-verifies.
- HTTP header names are case-insensitive, so a header set as `X-Session-Token` reads back lower-cased.

## See also

- [jwt-two-rsa-keypairs-link-and-session.md](jwt-two-rsa-keypairs-link-and-session.md) — the two key pairs behind the link and session tokens.
