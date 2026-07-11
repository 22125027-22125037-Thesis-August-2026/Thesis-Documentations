# Oracle Cloud (OCI) Rebuild Runbook — ⚠️ RETIRED

> **This runbook is no longer current.** uMatter was migrated off Oracle Cloud to **Microsoft Azure**
> on **2026-07-11**. The live runbook is [02-Azure-Cloud-Runbook](02-Azure-Cloud-Runbook.md).
>
> It is kept because it documents the platform the system ran on for most of its life, and because
> the parts that are *not* Oracle-specific — the 🔴 source-of-truth rule, the deploy-key/clone dance,
> the stack bring-up order — carried over unchanged and are explained more fully here.
>
> **What changed on Azure** (and why the steps below will mislead you): the VM is 8 GiB rather than
> 24 GiB, so the compose files now cap memory and JVM heap; Azure auto-shuts-down nightly, so every
> service declares a restart policy and the web UI is a systemd unit rather than tmux; and Azure has
> no host firewall, so the STEP 5 `iptables` work below is unnecessary.

---

<details>
<summary><b>Historical runbook (Oracle Cloud) — click to expand</b></summary>

> Step-by-step procedure to stand up the **entire uMatter backend** (4 Docker stacks / ~24
> containers + the web UI) on a fresh OCI VM, using secrets copied **from the laptop**.
>
> **Canonical source:** `D:\Y4-Sem 2 Thesis\Oracle deployment\OCl deployment from local.md`.

---

## 🔴 STEP 0 — The rule that makes this runbook work

`D:\Y4-Sem 2 Thesis` is the **single source of truth** for every non-GitHub file (`.env`, deploy
keys, `~/.ssh/config`, the Oracle VM SSH key, `notification_api/secrets/firebase-credentials.json`,
and future TLS/k8s/nginx files). Any such file changed on the VM **must** be copied back to the
laptop immediately. The VM is disposable; the laptop is not.

### Reference facts
| Thing | Value |
|---|---|
| VM name | `uMatter-dev` |
| Current public IP | `140.245.124.163` (changes every rebuild) |
| OS / user | Ubuntu 24.04 LTS / `ubuntu` |
| Shape | `VM.Standard.E5.Flex` — 4 OCPU / 24 GB (x86_64) |
| VM SSH key | `D:\Y4-Sem 2 Thesis\Oracle deployment\hieu-vm-oracle-ssh-key-2026-06-07.key` |
| GitHub deploy keys + config | `D:\Y4-Sem 2 Thesis\Oracle deployment\github-deploy-keys\` |
| GitHub org | `22125027-22125037-Thesis-August-2026` |

### Laptop repo ↔ VM directory ↔ GitHub repo map
| Laptop (`D:\Y4-Sem 2 Thesis\…`) | VM (`~/…`) | GitHub repo |
|---|---|---|
| `uMatter-Backend_Auth_Tracking_AI` *(a.k.a. `thesis-backend`)* | `uMatter-Backend_Auth_Tracking_AI` | `uMatter-Backend_Auth_Tracking_AI` (was `AuthenticationAPI`) |
| `notification-api` | `notification_api` | `notification_api` |
| `therapist-api` | `therapist-api` | `therapist-api` |
| `thesis_social` | `thesis_social` | `thesis_social` |
| `therapist-web-ui` | `therapist-web-ui` | `therapist-web-ui` (public, HTTPS clone) |

> ⚠️ Laptop dir names differ from VM dir names — mind this in every `scp`.

---

## STEP 1 — Provision the VM (networking FIRST, then the instance)

1. **New OCI account** (new email = fresh $300 / 30-day credit); pick a region with
   `VM.Standard.E5.Flex` capacity.
2. **Network first** — VCN Wizard → "Create VCN with Internet Connectivity": VCN `uMatter-VCN`,
   public subnet (public-IP assignment ON), internet gateway + route table.
3. **Security List ingress** (Source `0.0.0.0/0`, TCP): `80,443,8080-8087` (superset; the working
   minimum is `22,8080,8086,5173`) + `22`.
4. **SSH keypair** — generate on the Create-Instance page, **download the private key**, save as
   `D:\Y4-Sem 2 Thesis\Oracle deployment\ssh-key-<DATE>.key`, and update the reference table.
5. **Create instance** — name `uMatter-dev`, Ubuntu 24.04, shape `VM.Standard.E5.Flex` (4 OCPU/24 GB,
   x86 — *not* free ARM), boot 200 GB, public-IP toggle **ON**, into the public subnet.
6. Copy the public IP into the reference table, then SSH in:
   ```powershell
   ssh -i "D:\Y4-Sem 2 Thesis\Oracle deployment\ssh-key-<DATE>.key" ubuntu@<NEW_PUBLIC_IP>
   ```
   (If Windows rejects key perms: `icacls "<key>" /inheritance:r /grant:r "$($env:USERNAME):(R)"`.)

✅ Done when `ubuntu@umatter-dev:~$` appears.

---

## STEP 2 — Get code + secrets onto the VM (FROM THE LAPTOP)

### 2.1 Push deploy keys + ssh config (PowerShell on the laptop)
```powershell
$OR    = "ubuntu@<NEW_PUBLIC_IP>"
$ORKEY = "D:\Y4-Sem 2 Thesis\Oracle deployment\ssh-key-<DATE>.key"
$KEYS  = "D:\Y4-Sem 2 Thesis\Oracle deployment\github-deploy-keys"

scp -i "$ORKEY" "$KEYS\config" "$KEYS\AuthenticationAPI_deploy_key" `
  "$KEYS\id_ed25519_notification_api" "$KEYS\id_ed25519_notification_api.pub" `
  "$KEYS\id_ed25519_therapist_api"    "$KEYS\id_ed25519_therapist_api.pub" `
  "$KEYS\id_ed25519_thesis_social"    "$KEYS\id_ed25519_thesis_social.pub" `
  "${OR}:~/.ssh/"

ssh -i "$ORKEY" "$OR" "chmod 700 ~/.ssh; chmod 600 ~/.ssh/config ~/.ssh/AuthenticationAPI_deploy_key ~/.ssh/id_ed25519_*; chmod 644 ~/.ssh/*.pub 2>/dev/null; true"
```
> Perms matter: SSH ignores a group/world-readable private key. `config` + private keys → `600`,
> `.pub` → `644`, dir → `700`. The auth deploy key file is still named `AuthenticationAPI_deploy_key`
> even though the repo was renamed (deploy keys bind by repo ID).

### 2.2 Trust GitHub + verify each deploy key (on the VM)
```bash
ssh-keyscan -t rsa,ed25519 github.com >> ~/.ssh/known_hosts && chmod 600 ~/.ssh/known_hosts
for h in github-authapi github.com-notification_api github.com-therapist_api github.com-thesis_social; do
  ssh -T -o StrictHostKeyChecking=accept-new git@$h
done
```
Each prints `Hi <org>/<repo>! You've successfully authenticated…` ("no shell access" = success).

### 2.3 Clone the 5 repos (on the VM)
```bash
cd ~; ORG=22125027-22125037-Thesis-August-2026
git clone git@github-authapi:$ORG/uMatter-Backend_Auth_Tracking_AI.git uMatter-Backend_Auth_Tracking_AI
git clone git@github.com-notification_api:$ORG/notification_api.git notification_api
git clone git@github.com-therapist_api:$ORG/therapist-api.git therapist-api
git clone git@github.com-thesis_social:$ORG/thesis_social.git thesis_social
git clone https://github.com/$ORG/therapist-web-ui.git therapist-web-ui   # public, HTTPS
```

---

## STEP 3 — Drop the `.env` files + secrets (FROM THE LAPTOP)
```powershell
scp -i "$ORKEY" "D:\Y4-Sem 2 Thesis\thesis-backend\.env"   "${OR}:~/uMatter-Backend_Auth_Tracking_AI/.env"
scp -i "$ORKEY" "D:\Y4-Sem 2 Thesis\notification-api\.env" "${OR}:~/notification_api/.env"
scp -i "$ORKEY" "D:\Y4-Sem 2 Thesis\therapist-api\.env"    "${OR}:~/therapist-api/.env"
scp -i "$ORKEY" "D:\Y4-Sem 2 Thesis\thesis_social\.env"    "${OR}:~/thesis_social/.env"
scp -i "$ORKEY" "D:\Y4-Sem 2 Thesis\therapist-web-ui\.env"             "${OR}:~/therapist-web-ui/.env"
scp -i "$ORKEY" "D:\Y4-Sem 2 Thesis\therapist-web-ui\.env.development" "${OR}:~/therapist-web-ui/.env.development"
scp -i "$ORKEY" "D:\Y4-Sem 2 Thesis\notification-api\secrets\firebase-credentials.json" "${OR}:~/notification_api/secrets/firebase-credentials.json"
```
**IP swap** (on the VM) — `thesis_social/.env` and the three web-ui env files hard-code the IP:
```bash
OLD=161.118.252.10; NEW=<NEW_PUBLIC_IP>
sed -i "s/$OLD/$NEW/g" ~/thesis_social/.env ~/therapist-web-ui/.env ~/therapist-web-ui/.env.development
```
> `.env` files hold **zero port keys** (purged 2026-05; ports are literals in compose). A
> `${SOMETHING_PORT:-N}` creeping back in is a regression.

---

## STEP 4 — Confirm OCI Security List ingress
Confirm `80,443,8080-8087` (Source `0.0.0.0/0`, TCP) on the public subnet's security list. (Can't be
checked from inside the VM — use the OCI Console or the STEP 8 empirical test.)

## STEP 5 — Open Ubuntu's host firewall (iptables) — never flush
```bash
for p in 80 443 8080 8081 8083 5173; do
  sudo iptables -I INPUT 5 -m state --state NEW -p tcp --dport $p -j ACCEPT
done
sudo iptables -L INPUT -n --line-numbers      # confirm REJECT stays LAST
sudo netfilter-persistent save
```
> `5173` is the web UI (a host process), so it relies on this INPUT rule (Docker ports use FORWARD).

## STEP 6 — Install Docker + Compose
```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker ubuntu      # then open a NEW ssh session (or: newgrp docker)
docker --version && docker compose version
```

## STEP 6b — Create the shared external network (REQUIRED, before any compose up)
```bash
docker network create umatter-shared 2>/dev/null || echo "already exists"
```
Every stack declares `umatter-shared` as `external: true`, so none of them create it — `compose up`
fails with "network umatter-shared not found" unless it exists.

## STEP 7 — Bring up the Docker stacks (ORDER MATTERS)
```bash
cd ~/notification_api && docker compose up -d --build       # 1st
cd ~/therapist-api    && docker compose up -d --build       # 2nd
cd ~/thesis_social    && docker compose up -d --build       # 3rd
cd ~/uMatter-Backend_Auth_Tracking_AI && docker compose up -d --build   # 4th (heaviest)
```
Expect **20 running containers** here + `minio-init` (Exited 0). Only **one** pgAdmin (Auth's, `5051`).

## STEP 7b — Start the therapist-web-ui (NON-Docker, :5173)
```bash
# Node 20 if missing:
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt-get install -y nodejs
sudo apt-get install -y tmux
cd ~/therapist-web-ui && npm ci
tmux new-session -d -s web "cd ~/therapist-web-ui && npm run dev -- --host 0.0.0.0"
tmux ls                                  # -> web: 1 windows
sudo ss -ltnp | grep 5173                # confirm 0.0.0.0:5173
```
> Reattach: `tmux attach -t web` (detach `Ctrl-b d`). Does **not** auto-start on reboot — re-run.

---

## STEP 8 — Verify
```bash
docker ps -q | wc -l                                  # expect 24 (minio-init exits 0)
docker ps --format '{{.Names}}\t{{.Status}}' | sort
curl -s localhost:8080/health                         # nginx → healthy
curl -s localhost:8080/api/v1/dashboard/health        # nginx → dashboard → 200
for p in 8081 8082 8083 8084 8087; do
  printf 'port %s: ' $p; curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:$p/actuator/health
done
docker logs therapist-api-api-1 | grep Started        # therapist (8085) has no /actuator/health
docker logs social-api          | grep Started        # social    (8086) has no /actuator/health
```
From the laptop, prove the public path + both firewalls:
`Test-NetConnection <PUBLIC_IP> -Port 8080` and `Invoke-WebRequest "http://<PUBLIC_IP>:8080/health"` → `200 healthy`.

## STEP 9 — Repoint the clients at the new IP
1. **Mobile app** (`thesis-mobile`): `const BASE_URL = 'http://<NEW_PUBLIC_IP>:8080';`
2. **therapist-web-ui**: its env files were IP-swapped in STEP 3; if the IP changes again, re-run the
   `sed` and restart the tmux `web` session.

## STEP 10 — Decommission the old tenancy
Only after STEP 8 passes, delete the old account's paid resources (or let the trial lapse).

---

## Appendix — Quick rebuild checklist
1. New OCI account → region with E5.Flex capacity.
2. Network first: VCN + public subnet + IGW + route + security list `80,443,8080-8087`.
3. Create VM, public-IP ON, save the new SSH key to the laptop.
4. Push deploy keys + config from `github-deploy-keys\`.
5. Clone the 5 repos.
6. Copy `.env` + `secrets/` from the laptop; IP-swap social + web-ui.
7. iptables (incl. 5173) + Docker + Node 20.
8. `docker network create umatter-shared`, then `up -d` in order (notif → therapist → social → auth),
   then start the web UI in tmux.
9. Verify (STEP 8); repoint mobile + web UI (STEP 9).
10. 🔴 Keep the laptop updated with any new gitignored/secret files created on the VM thereafter.

</details>
