# Deployment

## Architecture

Two independently deployed services:

| Service | Platform | Cost |
|---|---|---|
| `client/` — Vite/React SPA | Cloudflare Workers (static assets) | Free |
| `server/` — Node.js WebSocket relay | Railway | ~$5/mo |

The server runs a persistent headless Chrome (Puppeteer) to intercept real-time lightning data from Blitzortung's WebSocket feed, then relays strikes and hotspot updates to the client over WebSocket. It also mirrors cloud imagery from an upstream source and serves it via HTTP. Cloudflare Workers cannot run Puppeteer or maintain persistent in-memory state, which is why the server lives on Railway.

The client connects to the server at runtime via `VITE_SERVER_URL`. HTTP and WebSocket both use this base URL (`http` → `ws` / `https` → `wss` swap is done in code).

---

## Prerequisites

- Railway account at railway.app (Hobby plan, $5/mo)
- Cloudflare account (free)
- `client/` pushed to a GitHub repo (`github.com/BenMiriello/overview-client`)
- `server/` pushed to a GitHub repo (`github.com/BenMiriello/overview-server`)

Both `client/` and `server/` are independent git repos (nested inside the root monorepo, which ignores them).

---

## First-Time Setup

### 1. Deploy the server on Railway

1. railway.app → **New Project** → **Deploy from GitHub repo** → select `overview-server`
2. Railway detects the `Dockerfile` and builds it automatically
3. After the first deploy: service → **Settings** (left sidebar) → **Networking** → **Public Networking** → **Generate Domain**
   - You'll get a URL like `https://overview-server-production.up.railway.app`
4. Set environment variables in the Railway dashboard under **Variables**:

| Variable | Production value |
|---|---|
| `VERBOSE` | `false` |
| `LOG_STREAM_DATA` | `false` |

Railway automatically injects `PORT`; the server reads it with `process.env.PORT || 3001`.

5. Note your Railway domain — you need it for the next step.

### 2. Deploy the client on Cloudflare Workers

The client deploys as a Cloudflare Worker with static assets (not Cloudflare Pages). The repo includes `wrangler.jsonc` which configures the deployment. SPA routing is handled by `not_found_handling: single-page-application` in that config — no `_redirects` file needed.

1. Cloudflare dashboard → **Workers & Pages** → **Create** → **Workers** → **Import from GitHub** → select `overview-client`
2. Build settings:
   - **Build command:** `npx vite build`
   - **Deploy command:** `npx wrangler deploy`
   - **Output directory:** `dist`
3. Under **Environment variables** → **Production**, add:

| Variable | Value |
|---|---|
| `VITE_SERVER_URL` | `https://overview-server-production.up.railway.app` |

4. Click **Save and Deploy**

### 3. Connect your custom domain

**Step A — Add domain to Cloudflare:**
1. Cloudflare dashboard → **Add a site** → enter your domain → select the Free plan
2. Cloudflare shows you two nameserver hostnames (e.g. `ada.ns.cloudflare.com`)

**Step B — Update Namecheap nameservers:**
1. Log in to Namecheap → find the domain → **Manage** → **Nameservers**
2. Select **Custom DNS** → enter the two Cloudflare nameservers
3. Wait for propagation (minutes to a few hours; check with `dig NS yourdomain.com`)

**Step C — Assign domain to the Worker:**
1. Cloudflare dashboard → **Workers & Pages** → select `overview-client`
2. **Settings** → **Domains & Routes** → **Add** → **Custom Domain** → enter your domain

Cloudflare issues a TLS certificate automatically.

---

## Re-deploying

| What changed | Action |
|---|---|
| Server code | Push to `overview-server` `main` branch — Railway redeploys automatically |
| Client code | Push to `overview-client` `main` branch — Cloudflare rebuilds and redeploys automatically |
| Server env vars | Update in Railway dashboard → Railway restarts the service |
| Client env vars | Update in Cloudflare Workers dashboard → trigger a new deployment |

---

## Environment Variables Reference

### Server (Railway)

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3001` | Injected by Railway automatically |
| `VERBOSE` | `false` | Enables verbose logging |
| `LOG_STREAM_DATA` | `false` | Logs raw WebSocket frame data |

### Client (Cloudflare Workers)

| Variable | Local (`.env`) | Production |
|---|---|---|
| `VITE_SERVER_URL` | `http://localhost:3001` | `https://overview-server-production.up.railway.app` |
| `VITE_SHOW_ATMOSPHERIC_LAYERS` | `false` | (not set — defaults to false) |

---

## Troubleshooting

**Puppeteer fails to launch on Railway**
- Check Railway build logs — Chrome deps must install cleanly from the Dockerfile
- The `PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium` env var must be set; it's baked into the Dockerfile
- If chromium crashes at runtime, check Railway service logs for signal 11 (segfault) — try adding `--disable-gpu` to Puppeteer launch args in `lightning_data.js`

**WebSocket connection fails in browser**
- Confirm `VITE_SERVER_URL` is set to the Railway `https://` URL (not `http://`)
- The client derives the WS URL by replacing `https` → `wss`; a plain `http` URL will produce `ws://` which browsers block from HTTPS pages (mixed content)
- Check browser devtools → Network → WS tab for the connection attempt and error

**Cloud images return 503**
- The server mirrors cloud images on startup and every 30 minutes; the cache at `server/cache/` is ephemeral on Railway (resets on redeploy)
- After a fresh deploy, allow ~30 seconds for the first mirror fetch to complete
- Check Railway logs for `[cloudMirror]` lines

**Strikes not appearing**
- Check Railway logs for `Navigating to Blitzortung map...` and `WebSocket opened` — if these don't appear, Puppeteer failed to launch or the page timed out
- Blitzortung sometimes rotates their WebSocket endpoints; a watchdog restart should recover within 2 minutes
