# Cloudflare setup for a Workers app: DNS, proxy, security, CDN

A Worker gets a real domain, edge security, and CDN behavior from one move — putting the domain's DNS zone in your own Cloudflare account; everything else hangs off that zone.

## Why it matters

Workers deploy fine to a `workers.dev` URL, but that hostname can't carry zone security rules (e.g. rate limiting), and a registrar-side CNAME pointing at `workers.dev` does not work — Cloudflare refuses hostnames it doesn't manage. The registrar's own "DNS powered by Cloudflare" (Porkbun offers this) is Cloudflare's network under the *registrar's* control, not your account, so it doesn't count either.

## How it works

**DNS.** Registration stays at the registrar; only nameservers change. Add the domain as a zone in your Cloudflare account (free plan), verify the imported DNS records match the registrar's, then swap nameservers at the registrar. Delete the old nameservers — resolvers pick any listed NS at random, so mixing old and new gives split-brain answers, and Cloudflare won't activate the zone until only its two remain.

**Proxy toggle.** Each record is either *Proxied* (orange cloud: Cloudflare terminates TLS and sits in the middle) or *DNS only* (grey cloud: pure phone book). Records pointing at an external host that manages its own TLS (e.g. a Vercel site) must be **DNS only** — proxying them puts Cloudflare in front of the other platform and can break its certificates. Identical records + grey cloud = zero-downtime nameserver move.

**Worker attachment.** One block in `wrangler.toml`:

```toml
routes = [
  { pattern = "app.example.com", custom_domain = true }
]
```

On deploy, Cloudflare creates the DNS record and TLS cert automatically; changing the pattern later cleans up the old record. Attaching a route **disables the `workers.dev` URL by default**.

**Security.** Zone-level WAF is free: one rate-limiting rule (per-IP, 10-second window, block), five custom rules, and a basic managed ruleset. Blocked requests never reach the Worker — this also protects the free tier's request quota, which in-Worker throttling can't (the request has already counted). Navigate *into the zone* → Security → WAF; the account-level WAF page is an Enterprise upsell.

**CDN.** Nothing to add: Worker static assets are served from the edge location nearest the visitor, and the Worker itself runs at the edge — there is no distant origin to cache in front of. The only centralized piece is D1 (one primary location).

## Gotchas

- Zone security may 403 automated HTTP clients while real browsers pass — verify "site is down" reports in a browser before debugging the Worker.
- Creating a D1 database or setting secrets is account-level and works before any zone exists; only the custom domain and WAF need the zone.

## See also

- `single-worker-api-plus-static-spa.md`
- `cloudflare-cron-floor-plus-app-gating.md`
