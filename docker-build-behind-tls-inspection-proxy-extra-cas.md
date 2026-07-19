# Building a container image behind a corporate TLS-inspection proxy

Behind a TLS-inspecting proxy, an image build that installs OS packages fails until you add your org's root CA to a directory the Dockerfile copies into the trust store.

## Why it matters

TLS interception re-signs the package mirror's certificate with an org root the container doesn't trust, so the build fails with "self-signed certificate in certificate chain". If the Dockerfile already copies an `extra-cas/` dir into the trust store (a no-op on clean networks), the fix is dropping one file.

## Steps

1. Try a plain build first — if it succeeds, your network isn't intercepting and you're done.
2. If it fails with the SSL error, identify the interception product from the certificate issuer:
   ```bash
   echo | openssl s_client -connect <mirror-host>:443 -servername <mirror-host> 2>/dev/null \
     | openssl x509 -noout -issuer
   # e.g. issuer=... CN=Gateway CA - <vendor> ...
   ```
3. Find that product's root CA file on disk (e.g. some corporate agents on macOS install it under `/Library/Application Support/<vendor>/`; each product documents its own path).
4. Copy it into the build context's CA directory — any filename ending in `.pem` or `.crt` works, and keep the directory `.gitignore`d:
   ```bash
   cp <ca-file> <context>/extra-cas/corp-root.pem
   ```
5. Rebuild.

## Gotchas

- The CA is baked in at build time only; the image keeps working after you leave the corporate network.
- Keep a tracked `.gitkeep` in the CA directory — git can't preserve the empty directory the Dockerfile's `COPY` needs without it.
- An image you built contains your org's CA, so don't push it to a shared registry.
