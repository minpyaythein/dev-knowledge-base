# Sign an invite link and the session it grants with two separate RSA key pairs

Use two distinct RSA key pairs — one signs the invite-link token, the other signs the follow-up session token — so a leak of one cannot forge the other.

## Why it matters

Sharing one key pair for both tokens would let a leaked session key forge invite links, and the reverse. The split also lets each service hold only the key it needs: the service that consumes links can verify them but cannot mint them.

## How it works

The link-issuing service signs the link token with the link private key; the consuming service verifies it with the link public key. On success the consuming service signs a session token with the session private key, and later verifies each request with the session public key. Keys are stored as DER bytes and rebuilt with `KeyFactory("RSA")`: private via `PKCS8EncodedKeySpec`, public via `X509EncodedKeySpec`. The link token has no `exp` — its validity is controlled by a database flag, not time. The session token carries `exp` (24h) and `iat`.

| token | signed by | verified by | expiry |
|-------|-----------|-------------|--------|
| link token | issuing service | consuming service | none (DB flag) |
| session token | consuming service | consuming service | `exp` 24h + `iat` |

## Example

Both sides claim `issuer`, `audience`, and a subject id; only the key spec and direction differ.

```java
// sign (private)
RSAPrivateKey priv = (RSAPrivateKey) KeyFactory.getInstance("RSA")
        .generatePrivate(new PKCS8EncodedKeySpec(keyBytes));
JWT.create().withIssuer(iss).withAudience(aud).withClaim("subId", id).sign(Algorithm.RSA256(priv));
// verify (public): X509EncodedKeySpec(pubBytes) → RSA256(pub) → JWT.require(...)
```

## Gotchas

- The link token never expires by `exp`; it stays valid until a newer link flips a reissue flag on the old record.
- Keys can be stored per environment, so each environment has its own pair.

## See also

- [spa-session-jwt-verified-per-controller.md](spa-session-jwt-verified-per-controller.md) — how the session token is carried and verified on each request.
