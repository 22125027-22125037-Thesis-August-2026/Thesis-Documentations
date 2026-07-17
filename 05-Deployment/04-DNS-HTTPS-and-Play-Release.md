# DNS, HTTPS & Google Play Release

> How uMatter went from a raw‑IP HTTP backend to a **domain + HTTPS** edge, and the state of the
> **Google Play** release of the mobile app. Added 2026‑07‑02; updated for the Azure migration
> 2026‑07‑11. Companion to [01-Deployment-Overview](01-Deployment-Overview.md) and
> [02-Azure-Cloud-Runbook](02-Azure-Cloud-Runbook.md).
>
> **The payoff of this work:** when the backend moved from Oracle to Azure, the **shipped mobile app
> needed no change and no resubmission** — it addresses the backend by domain, and the domain simply
> followed the VM. See §7.

---

## 1. Why this exists

Originally the mobile app talked to the gateway as `http://<PUBLIC_IP>:8080` — **plain HTTP to a raw
IP**. For a mental‑health app handling sensitive data of minors that is unacceptable, and Google Play's
**Data Safety** form requires "encrypted in transit." So the backend was put behind a **domain name +
TLS**, and the app's release builds were switched to HTTPS/WSS with cleartext disabled.

## 2. DNS — DuckDNS

A free dynamic‑DNS subdomain points at the VM:

| | |
|---|---|
| Domain | **`umatter-apcs.duckdns.org`** |
| Provider | [DuckDNS](https://www.duckdns.org) (account `apcsthesisteam@gmail.com`) |
| Points to | the Azure public IP (**`85.211.241.204`** today) |
| Auto‑update | a `systemd` timer on the VM (`duckdns.timer` → `duckdns.service`) re‑publishes the VM's public IP every 5 min, so the domain **self‑heals** when the IP changes |

The updater's token lives in `/etc/systemd/system/duckdns.service`; a backup of the units is under
`D:\Y4-Sem 2 Thesis\Oracle deployment\https\` (folder name is historical — the units are current).

**How it actually works — and the migration trap.** The updater calls
`…/update?domains=umatter-apcs&token=…&ip=` with **`ip=` deliberately empty**, which makes DuckDNS
adopt the **caller's source IP**. Two consequences:

- There is **nothing to type into the DuckDNS dashboard** when you migrate, and no token to change.
  Whichever VM runs the timer *owns* the domain.
- ⚠️ **If two VMs run the timer, the domain flaps between them every 5 minutes.** When migrating you
  must **disable `duckdns.timer` on the old VM first**, then enable it on the new one. This is easy
  to miss because the old VM keeps working perfectly right up until it silently steals the domain
  back.

## 3. HTTPS — Caddy reverse proxy

**Caddy** (Ubuntu apt package) runs on the host, terminates TLS on **:443**, and reverse‑proxies to the
existing Docker services. It obtains and auto‑renews a **Let's Encrypt** certificate (HTTP‑01 challenge
on :80). HTTP is `308`‑redirected to HTTPS.

Routes (`/etc/caddy/Caddyfile`, backed up at `Azure Deployment\Caddyfile`):

| Path | Upstream | Purpose |
|---|---|---|
| `/ws*` | `127.0.0.1:8086` (social‑api) | STOMP‑over‑WebSocket chat (`wss://…/ws`) |
| `/legal/*` | static files in `/var/www/umatter-legal` | Privacy policy page |
| `/api/*`, `/internal/*`, `/mhsa-media/*`, `/health` | `127.0.0.1:8080` (nginx gateway) | All REST API + media |
| everything else | static files in `/var/www/umatter-web` | **Therapist web UI** (SPA; unknown paths fall back to `index.html`) |

The API paths are matched **explicitly** so the SPA fallback can never swallow one. The web UI is
served at the **domain root — the same origin as the API**, which is what makes its requests
same‑origin (see §5).

**Firewall:** on Azure the **NSG is the only gate** — there is no host firewall (`iptables` is
`ACCEPT`, `ufw` inactive), unlike Oracle which needed both. The NSG must allow **80/443**: `80` for
the Let's Encrypt HTTP‑01 challenge, `443` for the app itself.

> **Do not start Caddy before DNS points at the new VM.** The HTTP‑01 challenge resolves the domain,
> so starting Caddy early fails against the old host and burns Let's Encrypt rate limit.

## 4. Media over HTTPS

Treasure Box / diary images are **presigned S3 (MinIO) URLs**. The tracking‑service builds them from
`S3_PUBLIC_ENDPOINT`, and the URL host is bound into the SigV4 signature (`X-Amz-SignedHeaders=host`) —
so the presign host must equal the host the client actually connects to.

`S3_PUBLIC_ENDPOINT` is **`https://umatter-apcs.duckdns.org`** — the **domain, not an IP**
(`uMatter-Backend_Auth_Tracking_AI/docker-compose.yml`, committed to `main`). The gateway's
`location /mhsa-media/` proxies to MinIO forwarding `Host $http_host`, so the signature validates
end‑to‑end: **Caddy → nginx → MinIO**.

Because the presign host is the domain, **media survived the Azure migration with no edit at all.**
Verified on Azure: `/mhsa-media/` returns MinIO's own `403 AccessDenied` XML to an unsigned request,
which proves the full proxy chain reaches MinIO with the host intact.

## 5. Mobile app: release vs dev

The app switches endpoints on `__DEV__`:

| | Dev / debug build | Release build |
|---|---|---|
| REST `BASE_URL` | `http://85.211.241.204:8080` | `https://umatter-apcs.duckdns.org` |
| Chat WebSocket | `ws://85.211.241.204:8086/ws` | `wss://umatter-apcs.duckdns.org/ws` |
| Cleartext (HTTP) | allowed (Metro) via `src/debug` network‑security override | **forbidden** — `usesCleartextTraffic=false` + no‑cleartext config |

So Metro/local development is unchanged, while the shipped `.aab` is HTTPS‑only. (Verified: the release
JS bundle contains the HTTPS URL and the dev raw‑IP branch is stripped.)

Only the **dev** column names an IP, so a migration touches only the debug build — **the shipped
`.aab` is unaffected.**

### The therapist web UI — served same‑origin (no longer an exception)

The web UI used to be the one client the domain could not rescue: a Vite **dev server** on
`:5173` addressing the VM by raw IP, so every migration meant repointing it. It is now a **static
build served by Caddy at the domain root**, i.e. the same origin as the API:

| | Before | Now |
|---|---|---|
| Served by | `npm run dev` (Vite dev server) on `:5173`, plain HTTP | Caddy `file_server` from `/var/www/umatter-web`, **HTTPS** |
| URL | `http://85.211.241.204:5173` | `https://umatter-apcs.duckdns.org` |
| API calls | `http://<raw‑IP>:8080` → **cross‑origin** (needed CORS) | `/api/v1/…` on its own origin → **no CORS at all** |
| On IP/domain change | repoint `.env` + `config.ts`, rebuild | **nothing** — URLs derive from `window.location` |

`src/lib/api/config.ts` defaults `API_BASE_URL`/`CHAT_WS_URL` to the page's own origin, so **no host is
baked into the bundle**. `VITE_API_BASE_URL`/`VITE_CHAT_WS_URL` override it only for genuinely
cross‑origin setups — notably `.env.development`, where `npm run dev` on a laptop hits the VM and *does*
rely on the gateway's CORS allow‑list (which permits `localhost`).

**Rebuilding/redeploying the web UI** (after a `git pull` on the VM):

```bash
cd ~/therapist-web-ui && npm ci && npm run build
sudo rsync -a --delete dist/ /var/www/umatter-web/
```

No Caddy reload is needed — it serves the directory directly.

> **The CORS allow‑list no longer names any public host** (`map $http_origin $cors_origin` in
> `nginx/nginx.conf`). It used to list the **Oracle** IP `140.245.124.163`, which the Azure migration
> missed — a *fourth* hard‑coded‑IP site beyond the three this runbook tracked. The map then fell
> through to `""`, `Access-Control-Allow-Origin` was omitted, and the VM‑hosted UI could not log in;
> only the `localhost` entry kept laptop dev working, which hid it. Now that the deployed UI is
> same‑origin, the map serves **only cross‑origin dev servers** (`localhost` / `127.0.0.1`). Do not add
> a public host back — that is the trap that created this. See the `--force-recreate` note in
> [02-Azure-Cloud-Runbook](02-Azure-Cloud-Runbook.md#step-5) before editing this file.

## 6. Public endpoints (single source of truth)

| What | URL |
|---|---|
| **Therapist web UI** | **`https://umatter-apcs.duckdns.org`** |
| API base (gateway) | `https://umatter-apcs.duckdns.org` |
| Health check | `https://umatter-apcs.duckdns.org/health` → `200 healthy` |
| Chat WebSocket | `wss://umatter-apcs.duckdns.org/ws` |
| Media (presigned) | `https://umatter-apcs.duckdns.org/mhsa-media/…` |
| Privacy policy | `https://umatter-apcs.duckdns.org/legal/privacy.html` |

The web UI and the API share one origin — the UI is the domain root, the API is `/api/v1/*` under it.
The old `http://85.211.241.204:5173` dev-server URL is **retired** (`umatter-web.service` disabled).

## 7. Migrating the HTTPS edge to a new VM

The order below is what was actually executed for the **Oracle → Azure cutover on 2026‑07‑11**. See
[02-Azure-Cloud-Runbook § STEP 7](02-Azure-Cloud-Runbook.md).

1. **Bring the new VM fully up first, and verify it internally** (all containers healthy, gateway
   answering on `localhost:8080`). DNS still points at the old VM, so production stays up throughout
   and the old VM remains a rollback.
2. **Open the NSG on 80/443** *before* touching DNS, or the cert will not issue.
3. **Surrender the domain on the OLD VM:** `sudo systemctl disable --now duckdns.timer`. Do this
   **first** — otherwise both VMs republish their IP every 5 min and the domain flaps (§2).
4. **Claim it on the NEW VM:** install the two units, `systemctl enable --now duckdns.timer`, then
   `systemctl start duckdns.service` → `OK`. Confirm with a real client:
   `curl -w '%{remote_ip}' https://umatter-apcs.duckdns.org/health`.
5. **Only now start Caddy** → it passes the HTTP‑01 challenge and issues the cert automatically.
6. **Repoint the therapist web UI** (§5) — the one client the domain does not cover.
7. `S3_PUBLIC_ENDPOINT` is already the **domain** (committed), so presigned media needs **no edit**.

**What did *not* need doing:** no DuckDNS dashboard change, no token change, no `.aab` rebuild, no
Play Store resubmission, and no `S3_PUBLIC_ENDPOINT` edit.

## 8. Google Play release (mobile app)

The teen mobile app is being published to **Google Play** for **Vietnam + South Korea**.

| Item | Value |
|---|---|
| Package | `com.thesisapp` (versionCode 1 / versionName 1.0) |
| Signing | Play App Signing; upload key = `umatter-upload` keystore (`D:\…\umatter-upload.keystore`) |
| Privacy policy | `https://umatter-apcs.duckdns.org/legal/privacy.html` |
| Category / audience | Health & Fitness · target **13+** (no under‑13) |
| Account type | **Personal** → Google mandates a **12‑tester / 14‑day closed test** before production |
| Full submission pack | `thesis-mobile/PLAY_STORE_RELEASE.md` (store listing EN+VI, Data Safety, content rating, reviewer login, step order) |
| Store assets | `thesis-mobile/playstore-assets/` (512 icon), `thesis-mobile/screenshots/` |

> The signed, HTTPS‑built `.aab` is at
> `thesis-mobile/android/app/build/outputs/bundle/release/app-release.aab`.
