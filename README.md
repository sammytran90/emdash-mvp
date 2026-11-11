# emdash-mvp

Minimal [EmDash](https://github.com/emdash-cms/emdash) 1.0 site from the official Cloudflare starter (`npm create emdash@latest -- --template cloudflare:starter`). It is an Astro app set up to run on Cloudflare Workers, with content in D1 and media in R2.

Nothing in this repository has been deployed, and no Cloudflare resources have been created. Names in `wrangler.jsonc` are placeholders. Secrets are not committed.

Requires Node.js 22.23.4 or later.

## Run locally

```bash
npm install
npx emdash secrets generate --write .env
npm run dev
```

The dev server prints a local URL, normally `http://localhost:4321/`.

- Public site: `http://localhost:4321/`
- Admin: `http://localhost:4321/_emdash/admin/`

`npm run dev` runs `astro dev`. The Cloudflare adapter uses a local D1 database and a local R2 bucket, so this does not need a Cloudflare login. `.env` is gitignored. It only needs `EMDASH_ENCRYPTION_KEY`, which encrypts plugin settings stored as secrets. Copy `.env.example` if you want the variable name without generating a key, but generate a real key before you save plugin secrets.

`npm run build` runs `astro build`. `npm run preview` serves that build locally.

## Admin login and first run

A new database has no users. The first visit to `/_emdash/admin/` redirects to the setup wizard.

1. **Set up your site.** Enter a site title and an optional tagline. Leave **Include sample content** selected to apply the Welcome post and About page from `seed/seed.json`, then continue.
2. **Create your account.** Enter your email and an optional name, then continue.
3. **Secure your account.** Register a passkey when the browser asks (Touch ID, Windows Hello, a device PIN, or a security key). EmDash creates this first user as the administrator. Open the dashboard.

After that, `/_emdash/admin/` shows the login page. Choose **Sign in** and approve the same passkey. There is no password. A fresh database sends `/_emdash/admin/` to the setup wizard. Once an administrator exists, the same URL sends anonymous visitors to `/_emdash/admin/login`.

`astro dev` also prints a development-only shortcut, `/_emdash/api/setup/dev-bypass?redirect=/_emdash/admin`. It applies the seed, creates an administrator at `dev@emdash.local`, and signs that user in. It returns 403 outside `astro dev`, so it is not a production login. Use the passkey wizard for any deployed site.

The first person to finish the wizard becomes the administrator. On a public URL, complete setup immediately after the site is reachable. Passkeys work only on the hostname where they were created. For a custom domain, run setup on that domain, or set `EMDASH_SITE_URL` to that origin before setup. When a passkey creates the first account, EmDash stores the email from the wizard without verifying it. Email invites and magic links need an email provider, which this starter does not configure.

Until setup finishes, the home page has no published posts. After setup, Posts and Pages are in the admin navigation. The Starter seed defines those collections, category and tag taxonomies, and the optional sample entries.

## Deploy to Cloudflare

Do not deploy until the account, resource names, and encryption key below are yours. `npm run deploy` runs `astro build && wrangler deploy`. The first deploy can create the named D1 database and R2 bucket if they do not already exist. This repository does not include a `database_id`. Wrangler writes one after that database exists. Authenticate with `npx wrangler login` in the account that should own the Worker. This repo has not been logged in or deployed.

### Bindings

Configured in `wrangler.jsonc`. The binding names must match `astro.config.mjs` (`d1({ binding: "DB" })` and `r2({ binding: "MEDIA" })`).

| Binding | Product | Placeholder name | Purpose |
| --- | --- | --- | --- |
| `DB` | D1 | database `emdash-mvp` | Content, users, and settings |
| `MEDIA` | R2 | bucket `emdash-mvp-media` | Uploaded media. The binding is the credential; no R2 access keys. |
| — | Cron Trigger | `* * * * *` | Scheduled publishing, plugin tasks, and backups. `src/worker.ts` exports the `scheduled` handler. |

`@astrojs/cloudflare` adds these during `astro build`. They are not listed in `wrangler.jsonc`:

| Binding | Product | Purpose |
| --- | --- | --- |
| `SESSION` | Workers KV | Astro sessions. Wrangler provisions the namespace on deploy. |
| `IMAGES` | Cloudflare Images | Resize and optimize media. |
| `ASSETS` | Workers Static Assets | Built front end and admin files. |

Sandboxed plugins are off. `worker_loaders` / `LOADER` stays commented out in `wrangler.jsonc`, and `astro.config.mjs` does not set `sandboxRunner`. That binding needs the Workers Paid plan. Leave it off unless you turn sandboxed plugins on later.

Change `name`, `database_name`, and `bucket_name` before the first deploy if these placeholders are not the resources you want. Keep `binding` values as `DB` and `MEDIA`.

### Secrets

Set Worker secrets with `npx wrangler secret put <NAME>`. Do not put values in `wrangler.jsonc`, `astro.config.mjs`, or `import.meta.env`.

| Secret | Required | Placeholder |
| --- | --- | --- |
| `EMDASH_ENCRYPTION_KEY` | Yes, before saving any encrypted plugin setting | Value of `npx emdash secrets generate`. Use the same key as local `.env` if this database already has encrypted settings. Keep a backup outside D1. |
| `EMDASH_SITE_URL` | Only when the public origin is known before setup | `https://example.com` (no trailing path). Can be a Wrangler var rather than a secret. |
| `EMDASH_PREVIEW_SECRET` | No | Omit to let EmDash store a generated preview HMAC secret in D1. |
| `EMDASH_IP_SALT` | No | Omit to let EmDash store a generated commenter-IP salt in D1. |

Optional later, not configured here: `EMAIL_SMTP_*` for the built-in SMTP transport, `CF_ACCESS_AUDIENCE` for Cloudflare Access, and `EMDASH_TURNSTILE_SECRET_KEY` for comment Turnstile. Magic-link login stays unavailable until an email provider is configured.

After deploy, open the `workers.dev` URL and finish `/_emdash/admin/` there if you did not set `EMDASH_SITE_URL` and run setup locally against that same database.

## Built-in MCP server

The site MCP server is enabled by default. `emdash()` in `astro.config.mjs` does not set `mcp: false`, so EmDash mounts it at `/_emdash/api/mcp`.

- Local: `http://localhost:4321/_emdash/api/mcp`
- Deployed: `https://<your-host>/_emdash/api/mcp`

The transport is stateless Streamable HTTP. Clients send JSON-RPC with `POST`. The endpoint requires a Bearer token. An admin browser session is not accepted. A request without a token is rejected with `401` and a `WWW-Authenticate: Bearer` challenge.

After the setup wizard:

1. In the admin, create a personal access token (prefix `ec_pat_`) under Settings, API tokens, and grant only the scopes that client needs. Or connect an interactive client with OAuth and approve scopes in the browser.
2. Point the client at the URL above and send `Authorization: Bearer <token>`.

To turn the server off, set `mcp: false` inside the `emdash({ ... })` options in `astro.config.mjs` and rebuild. That is separate from `.mcp.json`, `.cursor/mcp.json`, and `.vscode/mcp.json`, which point coding agents at the EmDash documentation MCP (`https://docs.emdashcms.com/mcp`) and do not manage this site's content.

## What this starter renders

| Page | Route |
| --- | --- |
| Home | `/` |
| All posts | `/posts` |
| Post | `/posts/:slug` |
| Category archive | `/category/:slug` |
| Tag archive | `/tag/:slug` |
| Page | `/:slug` |

Content is queried at runtime from D1. Uploaded files go to the `MEDIA` R2 bucket.

## References

- [EmDash on Cloudflare Workers](https://developers.cloudflare.com/workers/framework-guides/web-apps/emdash/)
- [Deploy to Cloudflare](https://docs.emdashcms.com/deployment/cloudflare/)
- [MCP server](https://docs.emdashcms.com/reference/mcp-server/)
- [AI tools](https://docs.emdashcms.com/guides/ai-tools/)
