# OpenSEO — open-source Semrush/Ahrefs alternative for you and your AI agent (v0.1.13). Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/TEMPLATE_CODE)

[OpenSEO](https://github.com/every-app/open-seo) (MIT) is a pay-as-you-go SEO tool: keyword research, rank tracking, competitor insights, backlinks, site audits, AI visibility tracking, shareable reports, and a built-in **MCP server** so Claude Code, Codex, ChatGPT, OpenClaw or Hermes can use your SEO data. There is no subscription. You bring your own [DataForSEO](https://dataforseo.com/) key and pay DataForSEO directly for what you use (the hosted openseo.so adds a 28% markup).

This template runs the **official v0.1.13 image** behind a password-protected gateway, with all data on a Railway volume.

## Why this template
- **Pinned:** `ghcr.io/every-app/open-seo:v0.1.13` and `caddy:2.11.7-alpine`, both pinned by tag **and** digest. Nothing uses `latest`.
- **Not open to the world:** OpenSEO's self-host auth modes are `cloudflare_access` (needs Cloudflare Zero Trust in front), `hosted` (the openseo.so SaaS mode: Google OAuth, Loops e-mail, and Autumn billing on every DataForSEO call), and `local_noauth`. Only `local_noauth` runs stand-alone, so the app service gets **no public domain**. A small Caddy **Gateway** is the only public entry point, and it asks for a username and a password that is generated per deploy (bcrypt-hashed at boot).
- **Zero deploy-time inputs:** the app boots and reports healthy without a DataForSEO key. Add `DATAFORSEO_API_KEY` whenever you're ready.
- **Persistent:** OpenSEO keeps everything (D1/SQLite database, KV, R2 objects, the SAM agent's Durable Object storage) in `/app/.wrangler`, which is mounted on a volume.
- **Scheduler included:** the image's supervisor runs rank-tracking and AI-visibility schedules every 5 minutes. No cron service needed.
- **MCP ready:** `https://<your-domain>/mcp` works through the gateway with HTTP Basic auth.

## Services
| Service | Source | Port | Public | Volume | Healthcheck |
|---|---|---|---|---|---|
| Gateway | `caddy:2.11.7-alpine@sha256:d8542f48d34a9cf4e4c11a478865229840e87e4c96ea3f439101f31a5d35f75f` | 8080 | **yes** (only public service) | none | `/healthz` (120 s) |
| OpenSEO | `ghcr.io/every-app/open-seo:v0.1.13@sha256:314ac3e08ee92a6b946d203d05421396918f66786678a030fbc62a64b877b8e8` | 3001 | no (private network only) | `/app/.wrangler` | `/api/health` (900 s) |

**OpenSEO** has no start command. The image CMD runs the env preflight, applies the D1 migrations, builds the app (~1–2 min, once per deploy), then serves on `0.0.0.0:3001`.

**Gateway** start command (also in [`gateway/start-command.txt`](gateway/start-command.txt)):
```sh
sh -c 'export OPENSEO_PASSWORD_HASH="$(caddy hash-password --plaintext "$OPENSEO_PASSWORD")" && printf "%s\n" "$CADDYFILE" > /etc/caddy/Caddyfile && exec caddy run --config /etc/caddy/Caddyfile --adapter caddyfile'
```
`CADDYFILE` holds [`gateway/Caddyfile`](gateway/Caddyfile). `/healthz` and `/api/health` are open. Everything else (UI, server functions, `/mcp`, `/agents` websockets) needs the login. Railway's `X-Forwarded-*` headers pass through, so OpenSEO builds correct `https://` links.

## Variables
### OpenSEO
| Variable | Template value | Notes |
|---|---|---|
| AUTH_MODE | `local_noauth` | safe here because the service is private; the Gateway does the auth |
| CLOUDFLARE_INCLUDE_PROCESS_ENV | `true` | passes Railway variables into the Workers runtime (from the official compose) |
| PORT | `3001` | |
| ALLOWED_HOST | `.railway.app` | vite host allow-list: covers `*.up.railway.app` and Railway's healthcheck host |
| BETTER_AUTH_SECRET | `${{secret(48)}}` | encrypts Google OAuth tokens once you add Search Console |
| VITE_SHOW_DEVTOOLS | `false` | from the official compose |
| DATAFORSEO_API_KEY | *(empty, you supply it)* | base64 of `login:api_password`, see below |

### Gateway
| Variable | Template value | Notes |
|---|---|---|
| PORT | `8080` | public domain targets this port |
| OPENSEO_USERNAME | `admin` | |
| OPENSEO_PASSWORD | `${{secret(24)}}` | **your login password** |
| OPENSEO_UPSTREAM | `${{OpenSEO.RAILWAY_PRIVATE_DOMAIN}}:${{OpenSEO.PORT}}` | |
| CADDYFILE | contents of `gateway/Caddyfile` | |

All values are also in [`env.example`](env.example).

## After deploy
1. Wait until both services are green. The first OpenSEO boot builds the app, which takes about 2 minutes. Until it finishes, the gateway shows "OpenSEO is starting".
2. Open **Gateway → Variables** and copy `OPENSEO_PASSWORD`. Then open `https://<gateway-domain>` and sign in as `admin` with that password.
3. Add your DataForSEO key. Create an API account at dataforseo.com, then run `printf 'you@example.com:API_PASSWORD' | base64` and set the output as `DATAFORSEO_API_KEY` on **OpenSEO**. It's the base64 of login:password, not the dashboard key. `https://<domain>/api/health` shows `"dataforseo":{"status":"ok"}` once it's set.
4. Optional: `OPENROUTER_API_KEY` turns on SAM (the AI SEO agent) and AI Visibility prompt ideas. `CONTEXT_API_KEY` turns on JS rendering in site audits. `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` add Search Console and Analytics (see the upstream docs).
5. Connect an agent: point it at `https://<gateway-domain>/mcp` and add the header `Authorization: Basic <base64 of admin:password>`. For example, `claude mcp add --transport http openseo https://<domain>/mcp --header "Authorization: Basic $(printf 'admin:PASSWORD' | base64)"`.

To change the password, edit `OPENSEO_PASSWORD` on Gateway. It's re-hashed on the next boot.

## Custom domain
Add the domain to **Gateway** (port 8080). Keep `ALLOWED_HOST=.railway.app`, because Railway's healthcheck needs it. Then set `BETTER_AUTH_URL=https://your.domain` on **OpenSEO**. In `local_noauth` mode that variable only adds a second host to the allow-list. Without it, vite answers "Blocked request" for the new hostname.

## Persistence
- **OpenSEO volume `/app/.wrangler`:** projects, keywords, rank history, audits, reports, AI-visibility data, files, and agent chat state.
- **Gateway** is stateless.
- A volume can only be mounted by one container, so a redeploy of OpenSEO causes about 2 minutes of downtime while the new container builds.

## Notes
- Telemetry: OpenSEO sends an anonymous usage heartbeat (counts only). Set `OPENSEO_TELEMETRY_DISABLED=1` on OpenSEO to turn it off.
- Resources: the app builds at boot (Node 22 + workerd). Memory peaks at about 4 GB during that build, then idles around 0.7 GB. The Gateway uses a few MB.
