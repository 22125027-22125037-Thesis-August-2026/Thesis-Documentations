# DNS, HTTPS & Google Play Release

> How uMatter went from a raw‑IP HTTP backend to a **domain + HTTPS** edge, and the state of the
> **Google Play** release of the mobile app. Added 2026‑07‑02. Companion to
> [01-Deployment-Overview](01-Deployment-Overview.md) and [02-Oracle-Cloud-Runbook](02-Oracle-Cloud-Runbook.md).

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
| Points to | the current OCI public IP (`140.245.124.163` today) |
| Auto‑update | a `systemd` timer on the VM (`duckdns.timer` → `duckdns.service`) re‑publishes the VM's public IP every 5 min, so the domain **self‑heals** when the IP changes on a rebuild |

The updater's token lives in `/etc/systemd/system/duckdns.service`; a backup of the units is under
`D:\Y4-Sem 2 Thesis\Oracle deployment\https\`.

## 3. HTTPS — Caddy reverse proxy

**Caddy** (Ubuntu apt package) runs on the host, terminates TLS on **:443**, and reverse‑proxies to the
existing Docker services. It obtains and auto‑renews a **Let's Encrypt** certificate (HTTP‑01 challenge
on :80). HTTP is `308`‑redirected to HTTPS.

Routes (`/etc/caddy/Caddyfile`, backed up at `Oracle deployment\https\Caddyfile`):

| Path | Upstream | Purpose |
|---|---|---|
| `/ws*` | `127.0.0.1:8086` (social‑api) | STOMP‑over‑WebSocket chat (`wss://…/ws`) |
| `/legal/*` | static files in `/var/www/umatter-legal` | Privacy policy page |
| everything else | `127.0.0.1:8080` (nginx gateway) | All REST API + `/mhsa-media/` |

**Firewall:** OCI Security List and host `iptables` already allow **80/443** (both were open on the
current VM). No firewall change was needed to activate Caddy.

## 4. Media over HTTPS

Treasure Box / diary images are **presigned S3 (MinIO) URLs**. The tracking‑service builds them from
`S3_PUBLIC_ENDPOINT`, and the URL host is bound into the SigV4 signature (`X-Amz-SignedHeaders=host`) —
so the presign host must equal the host the client actually connects to.

`S3_PUBLIC_ENDPOINT` was changed from `http://140.245.124.163:8080` to **`https://umatter-apcs.duckdns.org`**
(`uMatter-Backend_Auth_Tracking_AI/docker-compose.yml`, committed to `main` — commit "presign media over
the HTTPS domain"). The gateway's `location /mhsa-media/` proxies to MinIO forwarding `Host $http_host`, so
the signature validates end‑to‑end: **Caddy → nginx → MinIO**. Verified: a presigned URL returns
`200 image/jpeg`.

## 5. Mobile app: release vs dev

The app switches endpoints on `__DEV__`:

| | Dev / debug build | Release build |
|---|---|---|
| REST `BASE_URL` | `http://140.245.124.163:8080` | `https://umatter-apcs.duckdns.org` |
| Chat WebSocket | `ws://140.245.124.163:8086/ws` | `wss://umatter-apcs.duckdns.org/ws` |
| Cleartext (HTTP) | allowed (Metro) via `src/debug` network‑security override | **forbidden** — `usesCleartextTraffic=false` + no‑cleartext config |

So Metro/local development is unchanged, while the shipped `.aab` is HTTPS‑only. (Verified: the release
JS bundle contains the HTTPS URL and the dev raw‑IP branch is stripped.)

## 6. Public endpoints (single source of truth)

| What | URL |
|---|---|
| API base (gateway) | `https://umatter-apcs.duckdns.org` |
| Health check | `https://umatter-apcs.duckdns.org/health` → `200 healthy` |
| Chat WebSocket | `wss://umatter-apcs.duckdns.org/ws` |
| Media (presigned) | `https://umatter-apcs.duckdns.org/mhsa-media/…` |
| Privacy policy | `https://umatter-apcs.duckdns.org/legal/privacy.html` |

## 7. Rebuild notes (add to the runbook flow)

When rebuilding on a fresh OCI account (see [02-Oracle-Cloud-Runbook](02-Oracle-Cloud-Runbook.md)):

1. The DuckDNS **auto‑updater** re‑points `umatter-apcs.duckdns.org` at the new IP automatically once the
   `duckdns.timer` is installed — restore the two units from `Oracle deployment\https\`.
2. `sudo apt-get install -y caddy`, restore `Oracle deployment\https\Caddyfile` to `/etc/caddy/Caddyfile`,
   restore the privacy page to `/var/www/umatter-legal/`, then `systemctl enable --now caddy` (it re‑issues
   the cert automatically). Ensure OCI + iptables allow **80/443**.
3. `S3_PUBLIC_ENDPOINT` is already the **domain** (committed), so presigned media stays valid across IP
   changes — no per‑rebuild edit needed. **The app no longer hard‑codes the IP in release builds.**

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
