# Azure Rebuild Runbook

> Step-by-step procedure to stand up the **entire uMatter backend** (4 Docker stacks / 20 containers
> + the web UI + the HTTPS edge) on a fresh Azure VM, using secrets copied **from the laptop**.
>
> This replaces the Oracle runbook, which is retired but kept for history at
> [05-Oracle-Cloud-Runbook-(Retired)](05-Oracle-Cloud-Runbook-(Retired).md).
> Read [01-Deployment-Overview](01-Deployment-Overview.md) first.
>
> **Verified end-to-end on 2026-07-11** — including an unattended reboot test.

---

## 🔴 STEP 0 — The rule that makes this runbook work

`D:\Y4-Sem 2 Thesis` is the **single source of truth** for every file that is **not** committed to
GitHub. Any such file changed on the VM **must** be copied back to the laptop immediately. The VM is
disposable; the laptop is not.

The non-GitHub files, in full:

| File | Laptop location |
|---|---|
| VM SSH key | `Azure Deployment\Hieu_Azure_umatter-backend-vm_key.pem` |
| GitHub deploy keys + `config` | `Oracle deployment\github-deploy-keys\` *(name is historical; still current)* |
| `.env` × 5 | each repo root |
| Firebase credentials | `notification-api\secrets\firebase-credentials.json` |
| **Caddyfile** (serves the web UI + proxies the API) | `Azure Deployment\Caddyfile` — **current**; the older copy under `Oracle deployment\https\` predates the same-origin move |
| DuckDNS units | `Oracle deployment\https\` *(name is historical; still current)* |
| Privacy / legal pages | `Oracle deployment\legal\` |

### Reference facts

| Thing | Value |
|---|---|
| VM name | `umatter-backend-vm` |
| Public IP | **`85.211.241.204`** — **Static**, so it survives deallocation |
| Size | `Standard_B2as_v2` — 2 vCPU / 8 GiB (x86_64) |
| OS / user | Ubuntu 24.04 LTS / `azureuser` |
| Disk / swap | 29 GB OS disk + **8 GB swap file** (added; see STEP 3) |
| Resource group | `umatter-rg` |
| Subscription | `2957014a-9681-4ea5-8645-ad2d78ef1f73` *(Hieu's — **not** the `22125037@student` "Azure for Students" login)* |
| Region | Malaysia West |
| SSH | `ssh -i "D:\Y4-Sem 2 Thesis\Azure Deployment\Hieu_Azure_umatter-backend-vm_key.pem" azureuser@85.211.241.204` |
| GitHub org | `22125027-22125037-Thesis-August-2026` |

> If Windows rejects the key's permissions:
> `icacls "<key>" /inheritance:r /grant:r "$($env:USERNAME):(R)"`

### Laptop repo ↔ VM directory ↔ GitHub repo map

| Laptop (`D:\Y4-Sem 2 Thesis\…`) | VM (`~/…`) | GitHub repo |
|---|---|---|
| `uMatter-Backend_Auth_Tracking_AI` | `uMatter-Backend_Auth_Tracking_AI` | `uMatter-Backend_Auth_Tracking_AI` |
| `notification-api` | `notification_api` | `notification_api` |
| `therapist-api` | `therapist-api` | `therapist-api` |
| `thesis_social` | `thesis_social` | `thesis_social` |
| `therapist-web-ui` | `therapist-web-ui` | `therapist-web-ui` (public, HTTPS clone) |

> ⚠️ `notification-api` (laptop) ↔ `notification_api` (VM). Mind the hyphen in every `scp`.

---

## ⚠️ What Azure changes vs Oracle — read before you start

Four things bite differently here. Each one is a step below.

1. **The VM is 8 GiB, not 24 GiB.** Seven Spring Boot JVMs with no heap cap will each claim 25% of
   host RAM and overcommit the box several times over. The compose files now carry `mem_limit` +
   `MaxRAMPercentage` for every service — **this is committed to GitHub, so a fresh clone is already
   correct.** Do not remove them.
2. **Auto-shutdown is ON** (nightly, to conserve credit). Every service therefore declares
   `restart: unless-stopped`. Without this the VM comes back with the stack stopped. Also committed.
   (The web UI needs nothing here — it is static files served by Caddy, not a process.)
3. **There is no host firewall.** `iptables` is `ACCEPT` and `ufw` is inactive, unlike Oracle which
   needed both. The **NSG is the only gate** — nothing to configure inside the VM.
4. **The public IP should be Static.** A Dynamic IP is released on deallocation, so the nightly
   auto-shutdown hands you a new IP each morning. This used to *break the therapist web UI*, which
   talked to the raw IP; the UI is now served same-origin behind the DuckDNS domain, so a changing IP
   is merely churn (DuckDNS republishes within 5 min) rather than an outage. Keep it Static anyway.

---

## STEP 1 — Provision the VM

1. Ubuntu Server **24.04 LTS**, size **`Standard_B2as_v2`** (2 vCPU / 8 GiB). Do **not** take
   `B2als_v2` — the `l` is the low-memory variant (4 GiB) and the stack will not fit.
2. Save the private key to `Azure Deployment\` and update the reference table above.
3. **Set the public IP allocation to Static** (Portal → the VM's Public IP → Configuration, or):
   ```bash
   az network public-ip update -g umatter-rg -n <IP_NAME> --allocation-method Static
   ```
4. **NSG inbound rule** — TCP `80, 443, 8080, 8086` from `*` (plus `22`, which Azure adds by
   default). `5173` is no longer needed: the web UI is static files behind Caddy on `443`, not a Vite
   dev server:
   ```bash
   NSG=$(az network nsg list -g umatter-rg --query "[0].name" -o tsv)
   az network nsg rule create -g umatter-rg --nsg-name "$NSG" \
     -n umatter-web --priority 1010 --protocol Tcp --access Allow --direction Inbound \
     --source-address-prefixes '*' --destination-port-ranges 80 443 8080 8086
   ```
   `80` and `443` are **not optional**: Let's Encrypt needs `80` for the HTTP-01 challenge, and the
   Play Store app talks to `443`.

✅ Done when `azureuser@umatter-backend-vm:~$` appears.

> **Verifying a port is genuinely open:** a probe against a port with **nothing listening** fails
> exactly like an NSG block. To test the NSG itself, put a real listener on the port first
> (`sudo python3 -m http.server 80`) and probe from the laptop. Otherwise you will misread "Caddy is
> stopped" as "the firewall is closed."

---

## STEP 2 — Host prep: swap, Docker, Node

```bash
# 8 GB swap — headroom for image builds; the running stack should never touch it.
sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab
echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-swap.conf && sudo sysctl -w vm.swappiness=10

# Docker + Compose
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg tmux
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker azureuser      # then open a NEW ssh session

# Node 20 (for the web UI)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt-get install -y nodejs

# The shared network EVERY stack expects (external: true — no stack creates it)
docker network create umatter-shared
```

---

## STEP 3 — Deploy keys, then clone (FROM THE LAPTOP)

```powershell
$AZ    = "azureuser@85.211.241.204"
$AZKEY = "D:\Y4-Sem 2 Thesis\Azure Deployment\Hieu_Azure_umatter-backend-vm_key.pem"
$KEYS  = "D:\Y4-Sem 2 Thesis\Oracle deployment\github-deploy-keys"

scp -i "$AZKEY" "$KEYS\config" "$KEYS\AuthenticationAPI_deploy_key" `
  "$KEYS\id_ed25519_notification_api" "$KEYS\id_ed25519_notification_api.pub" `
  "$KEYS\id_ed25519_therapist_api"    "$KEYS\id_ed25519_therapist_api.pub" `
  "$KEYS\id_ed25519_thesis_social"    "$KEYS\id_ed25519_thesis_social.pub" `
  "${AZ}:~/.ssh/"
```

On the VM — perms matter (SSH silently ignores a group-readable private key):

```bash
chmod 700 ~/.ssh; chmod 600 ~/.ssh/config ~/.ssh/AuthenticationAPI_deploy_key ~/.ssh/id_ed25519_*
chmod 644 ~/.ssh/*.pub
ssh-keyscan -t rsa,ed25519 github.com >> ~/.ssh/known_hosts && chmod 600 ~/.ssh/known_hosts

for h in github-authapi github.com-notification_api github.com-therapist_api github.com-thesis_social; do
  ssh -T -o StrictHostKeyChecking=accept-new git@$h      # "Hi <org>/<repo>!" = success
done

cd ~; ORG=22125027-22125037-Thesis-August-2026
git clone git@github-authapi:$ORG/uMatter-Backend_Auth_Tracking_AI.git uMatter-Backend_Auth_Tracking_AI
git clone git@github.com-notification_api:$ORG/notification_api.git notification_api
git clone git@github.com-therapist_api:$ORG/therapist-api.git therapist-api
git clone git@github.com-thesis_social:$ORG/thesis_social.git thesis_social
git clone https://github.com/$ORG/therapist-web-ui.git therapist-web-ui
```

---

## STEP 4 — Secrets + the IP swap (FROM THE LAPTOP)

```powershell
scp -i "$AZKEY" "D:\Y4-Sem 2 Thesis\uMatter-Backend_Auth_Tracking_AI\.env" "${AZ}:~/uMatter-Backend_Auth_Tracking_AI/.env"
scp -i "$AZKEY" "D:\Y4-Sem 2 Thesis\notification-api\.env"                "${AZ}:~/notification_api/.env"
scp -i "$AZKEY" "D:\Y4-Sem 2 Thesis\therapist-api\.env"                   "${AZ}:~/therapist-api/.env"
scp -i "$AZKEY" "D:\Y4-Sem 2 Thesis\thesis_social\.env"                   "${AZ}:~/thesis_social/.env"
scp -i "$AZKEY" "D:\Y4-Sem 2 Thesis\therapist-web-ui\.env"                "${AZ}:~/therapist-web-ui/.env"
scp -i "$AZKEY" "D:\Y4-Sem 2 Thesis\notification-api\secrets\firebase-credentials.json" "${AZ}:~/notification_api/secrets/firebase-credentials.json"
```

Three files hard-code the IP. If the IP ever changes, re-run this on the VM:

```bash
OLD=<PREVIOUS_IP>; NEW=85.211.241.204
sed -i "s/$OLD/$NEW/g" ~/thesis_social/.env ~/therapist-web-ui/.env ~/therapist-web-ui/.env.development
```

- `thesis_social/.env` → `SOCIAL_CORS_ALLOWED_ORIGINS`
- `therapist-web-ui/.env` → the 5 `VITE_*_BASE_URL`s
- `therapist-web-ui/.env.development` → **git-tracked**, so a local `sed` will conflict with the next
  `git pull`. Prefer committing the change; if you have already `sed`-ed it, run
  `git checkout -- .env.development` before pulling.

> `S3_PUBLIC_ENDPOINT` needs **no** edit — it is the **domain**, not an IP, and is committed.

---

## STEP 5 — Bring up the stacks (ORDER MATTERS)

```bash
export COMPOSE_PARALLEL_LIMIT=1      # 2 vCPU: parallel Gradle builds thrash the box
cd ~/notification_api                 && docker compose up -d --build   # 1st
cd ~/therapist-api                    && docker compose up -d --build   # 2nd
cd ~/thesis_social                    && docker compose up -d --build   # 3rd
cd ~/uMatter-Backend_Auth_Tracking_AI && docker compose up -d --build   # 4th (heaviest: 4 services)
```

≈12 minutes for all seven Spring Boot images on 2 vCPU. Expect **20 running** + `minio-init`
(Exited 0 — it is a one-shot bucket initialiser and is the **only** service without a restart policy).

---

## STEP 6 — Build the web UI (static, served by Caddy)

The UI is a **static build** served by Caddy at the domain root — no long-running process, so nothing
to keep alive across the nightly auto-shutdown:

```bash
cd ~/therapist-web-ui && npm ci && npm run build
sudo mkdir -p /var/www/umatter-web
sudo rsync -a --delete dist/ /var/www/umatter-web/
```

Caddy's site block (STEP 7) serves `/var/www/umatter-web` at the root and proxies `/api/*` to the
gateway, so the UI and API share one origin — **no CORS**. Do not set `VITE_API_BASE_URL` in `.env`
for this build: leaving it unset makes the app derive its URLs from `window.location`, so no host is
baked in.

> **Superseded:** the UI used to run as a Vite dev server (`umatter-web.service` → `:5173`). That unit
> is disabled; `Azure Deployment\umatter-web.service` is kept only for historical reference.

---

## STEP 7 — The HTTPS edge (DuckDNS + Caddy)

**Order matters.** If the old VM is still alive, its `duckdns.timer` re-publishes *its* IP every 5
minutes and the domain will **flap** between the two hosts. Kill it first:

```bash
# 1. ON THE OLD VM (if it still runs) — surrender the domain:
sudo systemctl disable --now duckdns.timer

# 2. ON THE NEW VM — claim the domain:
sudo cp duckdns.service duckdns.timer /etc/systemd/system/     # from Oracle deployment\https\
sudo systemctl daemon-reload && sudo systemctl enable --now duckdns.timer
sudo systemctl start duckdns.service                            # -> "OK"
```

The updater posts `…&ip=` **empty**, so DuckDNS adopts the **caller's** source IP. There is nothing
to type into the DuckDNS dashboard, and no token to change — whichever VM runs the timer owns the
domain. It also means the domain **self-heals** whenever the IP changes.

Then Caddy (terminates TLS on `:443`, auto-renews via Let's Encrypt, `308`-redirects `:80`):

```bash
sudo apt-get install -y caddy
sudo cp Caddyfile /etc/caddy/Caddyfile                          # from Oracle deployment\https\
sudo mkdir -p /var/www/umatter-legal && sudo cp privacy.html delete-account.html /var/www/umatter-legal/
sudo chown -R caddy:caddy /var/www/umatter-legal
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl enable --now caddy
```

> **Do not start Caddy before DNS points at this VM** — the HTTP-01 challenge resolves the domain, so
> it will fail against the wrong host and burn Let's Encrypt rate limit.

---

## STEP 8 — Verify

```bash
docker ps -q | wc -l                                   # 20
curl -s localhost:8080/health                          # healthy
for p in 8081 8082 8083 8084 8087; do
  printf 'port %s: ' $p; curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:$p/actuator/health
done
docker logs therapist-api-api-1 | grep Started         # 8085 has no /actuator/health
docker logs social-api          | grep Started         # 8086 has no /actuator/health
docker stats --no-stream --format "{{.Name}}\t{{.MemUsage}}"   # every JVM well under its 640 MiB cap
```

From the laptop — the public surface the shipped app actually uses:

```bash
D=umatter-apcs.duckdns.org
curl -s https://$D/health                              # healthy
curl -s -o /dev/null -w "%{http_code}\n" https://$D/legal/privacy.html          # 200
curl -s -o /dev/null -w "%{http_code}\n" http://$D/health                       # 308 -> https
curl -s -o /dev/null -D- --http1.1 https://$D/ws \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ=="  # 101
```

> Two known-good oddities, both **pre-existing and harmless**: `/ws/info` returns **500** (a SockJS
> endpoint the app does not use — it opens a native WebSocket), and `/mhsa-media/` returns **403
> AccessDenied** to an unsigned request (that is MinIO answering, which proves the proxy chain works).
> Use `--http1.1` for the WebSocket check: over HTTP/2, curl cannot upgrade and reports a bogus 400.

**The reboot test — do not skip it.** It is the only thing that proves the stack survives the nightly
auto-shutdown:

```bash
sudo systemctl reboot
# wait ~2 min, touch nothing, then:
docker ps -q | wc -l                                   # 20, unattended
for s in docker caddy duckdns.timer; do systemctl is-active $s; done   # all active
curl -s https://umatter-apcs.duckdns.org/health        # healthy
```

---

## STEP 9 — Repoint the clients

| Client | Change needed |
|---|---|
| **Mobile — release build** (the Play Store `.aab`) | **None.** It resolves `https://umatter-apcs.duckdns.org`, which follows the VM. **No rebuild, no resubmission.** |
| **Mobile — debug build** | `__DEV__` branches in `src/api/axiosClient.ts` and `src/screens/social/ChatScreen.tsx` hard-code the raw IP. Metro-only; does not affect the shipped app. |
| **Therapist web UI** | **Required.** It talks to the **raw IP**, not the domain (Caddy only fronts `8080` and `/ws`), so the domain cannot carry it across. See the STEP 4 `sed`. |

---

## Appendix — Quick rebuild checklist

1. Ubuntu 24.04, **`Standard_B2as_v2`** (2 vCPU / **8 GiB** — not the 4 GiB `B2als_v2`).
2. Public IP → **Static**. NSG inbound → `80,443,8080,8086`.
3. Swap 8 GB, Docker, Node 20, `docker network create umatter-shared`.
4. Deploy keys → clone 5 repos → `.env` + Firebase secret → IP-swap social + web-ui.
5. `COMPOSE_PARALLEL_LIMIT=1`, then `up -d --build` in order (notif → therapist → social → auth).
6. Web UI: `npm run build` → rsync `dist/` to `/var/www/umatter-web` (Caddy serves it).
7. HTTPS: **disable the old VM's `duckdns.timer` first**, then DuckDNS + Caddy on the new one.
8. Verify (STEP 8) — **including the reboot test**.
9. Repoint the web UI. The mobile release build needs nothing.
10. 🔴 Copy any new non-GitHub file back to the laptop.
