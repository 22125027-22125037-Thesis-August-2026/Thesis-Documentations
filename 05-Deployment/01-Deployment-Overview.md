# Deployment Overview

> The big picture of how uMatter runs in production, before the step-by-step runbook. For the full
> rebuild procedure see [02-Oracle-Cloud-Runbook](02-Oracle-Cloud-Runbook.md); for laptop dev see
> [03-Local-Development-Setup](03-Local-Development-Setup.md).

---

## 1. Where it runs

uMatter is deployed on a **single Oracle Cloud Infrastructure (OCI) VM**:

| Thing | Value |
|---|---|
| VM name | `uMatter-dev` |
| Current public IP | `140.245.124.163` *(changes on every rebuild)* |
| OS | Canonical Ubuntu 24.04 LTS, user `ubuntu` |
| Shape | `VM.Standard.E5.Flex` — 4 OCPU / 24 GB RAM (AMD x86_64) |
| Boot volume | 200 GB |
| SSH | `ssh -i "D:\Y4-Sem 2 Thesis\Oracle deployment\hieu-vm-oracle-ssh-key-2026-06-07.key" ubuntu@140.245.124.163` |

The VM was chosen to fit inside Oracle's **$300 / 30-day free-trial** credit. When the trial lapses,
the VM is destroyed and rebuilt on a fresh account — which is exactly why the rebuild runbook and the
🔴 source-of-truth rule (below) exist.

---

## 2. What runs on the VM

```
                       OCI VM (Ubuntu 24.04, 4 OCPU / 24 GB)
   ┌────────────────────────────────────────────────────────────────────┐
   │  Docker Engine + Compose                                            │
   │                                                                    │
   │  Stack 1: notification_api  → notification-service, postgres, redis│
   │  Stack 2: therapist-api     → therapist-api, postgres              │
   │  Stack 3: thesis_social     → social-api, postgres, rabbitmq       │
   │  Stack 4: uMatter-Backend   → nginx(:8080), auth/ai/tracking/      │
   │                               dashboard, postgres×3, pgadmin,      │
   │                               redis, rabbitmq, minio, minio-init   │
   │                                                                    │
   │  All stacks join the `umatter-shared` external Docker network      │
   │                                                                    │
   │  Host process (NOT Docker): therapist-web-ui Vite server :5173     │
   │                              (in a tmux session)                   │
   └────────────────────────────────────────────────────────────────────┘
```

- **~24 containers** total across the 4 stacks (+ `minio-init`, which exits 0 by design).
- The **therapist web UI** is the only non-Docker piece — a Vite dev server in tmux on `:5173`.

---

## 3. Network exposure

Two firewall layers gate inbound traffic:

1. **OCI Security List** (cloud firewall) — the source of truth for the internet edge.
2. **Ubuntu iptables** (`iptables-persistent`) — host firewall; mainly covers host-bound access since
   Docker-published ports traverse the FORWARD chain.

**Actually open to the internet:** `22` (SSH), `8080` (gateway), `8086` (Social direct/STOMP-WS),
`5173` (web UI). Everything the mobile app needs flows through `8080`. (Optionally `5051` for
pgAdmin, otherwise SSH-tunnel it.) See [01-Architecture/02-Service-Catalog-and-Ports](../01-Architecture/02-Service-Catalog-and-Ports.md).

---

## 4. Bring-up order (matters)

```
docker network create umatter-shared          # once, before anything
cd ~/notification_api   && docker compose up -d --build   # 1st (consumers attach after)
cd ~/therapist-api      && docker compose up -d --build   # 2nd
cd ~/thesis_social      && docker compose up -d --build   # 3rd
cd ~/uMatter-Backend_…  && docker compose up -d --build   # 4th (heaviest: 4 Spring services)
# then the web UI:
tmux new-session -d -s web "cd ~/therapist-web-ui && npm run dev -- --host 0.0.0.0"
```

---

## 5. 🔴 The source-of-truth rule (read this)

> **`D:\Y4-Sem 2 Thesis` on the laptop is the SINGLE SOURCE OF TRUTH for every file that is NOT
> pushed to GitHub** — `.env` files, GitHub deploy keys + `~/.ssh/config`, the Oracle VM SSH key,
> notification's `secrets/firebase-credentials.json`, and anything added later (TLS certs/keys,
> Kubernetes manifests, custom nginx config, …).
>
> **Whenever such a file is created/modified/deleted on the VM, immediately copy that change back to
> the matching place under `D:\Y4-Sem 2 Thesis`.**

**Why:** these files never reach GitHub. When the Oracle trial credit runs out, the VM is gone and
you may not be able to log in to copy anything. The laptop copy is then the *only* surviving copy — it
is what makes a future rebuild possible (secrets come **from the laptop**, not from a dead VM).

---

## 6. Disaster recovery = the rebuild runbook

There is no live backup of the VM. Recovery means **rebuilding from scratch** on a new OCI account
using secrets from the laptop. That entire procedure — provision VM, push deploy keys, clone repos,
drop `.env` + secrets, open firewalls, install Docker + Node, create the shared network, bring up the
stacks, start the web UI, verify, repoint the clients — is documented step-by-step in
[02-Oracle-Cloud-Runbook](02-Oracle-Cloud-Runbook.md).

> The authoritative, original runbook lives at
> `D:\Y4-Sem 2 Thesis\Oracle deployment\OCl deployment from local.md`. The runbook page here is a
> navigable summary of it; if they ever diverge, treat the original as canonical and update both.

---

## 7. Data persistence

Postgres, MinIO, and pgAdmin use Docker named volumes. A rebuild starts with a **fresh seed** (live
data was not migrated on the 2026-06-07 dev→prod move). To carry real data forward: `pg_dump`/restore
per database + copy MinIO buckets.
