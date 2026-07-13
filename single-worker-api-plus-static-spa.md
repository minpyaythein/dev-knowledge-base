# One Cloudflare Worker serving API + SPA + cron — the $0 full-stack shape

A single Worker can be the whole app — REST API (Hono), the built React bundle as static assets, and scheduled background jobs — which keeps hosting at $0 and deployment to one command.

## Why it matters

The default mental model is three deployables: an API server, a static-site host, and a job runner. On Cloudflare's free tier one Worker covers all three, with no CORS (same origin), no server to patch, and one `wrangler deploy`. For hobby/portfolio-scale apps this is the cheapest serious architecture available.

## How it works

The Worker exports two handlers:

```ts
export default {
  fetch: app.fetch,          // Hono app: /api/* routes; static assets serve the SPA
  async scheduled(controller, env, ctx) {
    ctx.waitUntil(runJobs(env));   // cron trigger from wrangler.toml
  },
};
```

- **Static assets**: `wrangler.toml` points the assets config at the SPA's build output (e.g., `web/dist`). Requests that don't match an API route fall through to the static files. The SPA can use hash routing (`#/settings`) to avoid needing server-side route fallbacks entirely.
- **Build coupling**: the deploy/dev scripts must build the frontend first (`npm run build:web && wrangler dev`) — the Worker serves whatever is in `dist`, so a stale bundle silently shows old UI. Encode the ordering in package.json scripts, not in memory.
- **State**: D1 (SQLite) binds into `env`; both the API handlers and the cron jobs use the same DB binding — no connection strings.
- **Secrets**: `wrangler secret put` in production; a git-ignored `.env` locally (wrangler 4.x loads it natively). Same code path reads `env.NAME` in both.
- **`ctx.waitUntil`** keeps the cron work alive after the handler returns — without it, async work may be killed mid-flight.

## Gotchas

- Cron does **not** auto-fire under `wrangler dev`. Test scheduled handlers via `curl "http://localhost:8787/__scheduled?cron=…"`.
- The frontend can be its own npm package (own `package.json`/lockfile) inside the repo — Worker deps and React deps stay untangled.
