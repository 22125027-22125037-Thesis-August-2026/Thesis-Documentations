# Deployment Overview

> The big picture of how uMatter runs in production, before the step-by-step runbook. For the full
> rebuild procedure see [02-Azure-Cloud-Runbook](02-Azure-Cloud-Runbook.md); for laptop dev see
> [03-Local-Development-Setup](03-Local-Development-Setup.md).

---

## 1. Where it runs

uMatter is deployed on a **single Microsoft Azure VM** (migrated from Oracle Cloud on **2026-07-11**;
the retired Oracle runbook is kept at
[05-Oracle-Cloud-Runbook-(Retired)](05-Oracle-Cloud-Runbook-(Retired).md)):

| Thing | Value |
|---|---|
| VM name | `umatter-backend-vm` |
| Public IP | **`85.211.241.204`** — **Static** (survives deallocation) |
| Public HTTPS endpoint | **`https://umatter-apcs.duckdns.org`** *(stable domain via DuckDNS + Caddy — see [04-DNS-HTTPS-and-Play-Release](04-DNS-HTTPS-and-Play-Release.md))* |
| OS | Ubuntu 24.04 LTS, user `azureuser` |
| Size | `Standard_B2as_v2` — **2 vCPU / 8 GiB** (x86_64), burstable B-series |
| Disk | 29 GB OS disk + an **8 GB swap file** |
| Resource group / region | `umatter-rg` / Malaysia West |
| SSH | `ssh -i "D:\Y4-Sem 2 Thesis\Azure Deployment\Hieu_Azure_umatter-backend-vm_key.pem" azureuser@85.211.241.204` |

Funded by the **GitHub Student Developer Pack ($100 credit)**, with **nightly auto-shutdown** to
stretch it. That auto-shutdown is the single most important fact about this deployment — see §5.

> ⚠️ The VM lives in **Hieu's** Azure subscription (`2957014a-…`), **not** the
> `22125037@student.hcmus.edu.vn` "Azure for Students" subscription. Resizes and NSG rules must be
> made from that account.

---

## 2. What runs on the VM

```
                  Azure VM (Ubuntu 24.04, 2 vCPU / 8 GiB + 8 GB swap)
   ┌────────────────────────────────────────────────────────────────────┐
   │  Caddy (host, :80/:443) ── TLS termination, Let's Encrypt          │
   │      │                                                             │
   │      ├── /ws*  ────────────────► social-api      :8086             │
   │      ├── /legal/* ─────────────► /var/www/umatter-legal            │
   │      └── everything else ──────► nginx gateway   :8080             │
   │                                                                    │
   │  Docker Engine + Compose                                           │
   │   Stack 1: notification_api  → notification-service, postgres, redis│
   │   Stack 2: therapist-api     → therapist-api, postgres             │
   │   Stack 3: thesis_social     → social-api, postgres, rabbitmq      │
   │   Stack 4: uMatter-Backend   → nginx(:8080), auth/ai/tracking/     │
   │                                dashboard, postgres×3, pgadmin,     │
   │                                redis, rabbitmq, minio, minio-init  │
   │                                                                    │
   │   All stacks join the `umatter-shared` external Docker network     │
   │                                                                    │
   │  systemd: umatter-web  → therapist-web-ui (Vite) :5173             │
   │  systemd: duckdns.timer → republishes the public IP every 5 min    │
   └────────────────────────────────────────────────────────────────────┘
```

- **20 running containers** across the 4 stacks, plus `minio-init` (exits 0 by design).
- **7 Spring Boot JVMs**, 6 Postgres, 2 RabbitMQ, 2 Redis, MinIO, pgAdmin, nginx.
- The **therapist web UI** is the only non-Docker piece — a Vite server run by **systemd**
  (`umatter-web.service`), *not* tmux.

---

## 3. Network exposure

Unlike Oracle, **Azure has no host firewall to configure**: `iptables` is `ACCEPT` and `ufw` is
inactive. The **NSG (Network Security Group) is the only gate.**

Open to the internet: `22` (SSH), **`443`/`80` (Caddy HTTPS edge — the public entry point)**,
`8080` (gateway, behind Caddy), `8086` (social direct / STOMP-WS), `5173` (web UI).

> **Testing whether a port is open:** a probe against a port with **nothing listening** fails exactly
> like a firewall block. Put a real listener on it first (`sudo python3 -m http.server 80`) before
> concluding the NSG is closed.

---

## 4. The memory budget (why the compose files cap everything)

The Oracle VM had **24 GiB**; this one has **8 GiB**. A Spring Boot JVM with no `-Xmx` sizes its heap
at **25% of host RAM** — so seven of them lay claim to far more than the host has. Nothing in the
compose files capped anything, because 24 GiB hid the problem.

Every service now declares a `mem_limit`, and each JVM derives its heap from that limit via
`-XX:MaxRAMPercentage=60` (the JVM reads its own cgroup, so the two can never drift apart). This is
**committed to GitHub** — a fresh clone is already correct.

Measured in production, at rest:

| | |
|---|---|
| Host RAM used | **~3.3–4.0 GiB / 7.8 GiB** |
| Each Spring Boot JVM | ~290–345 MiB against a 640 MiB cap (~50%) |
| Swap used | **0** |
| OOM kills | **0** |

pgAdmin is the only container that runs near its ceiling and nothing depends on it — it is the first
thing to `docker compose stop` under memory pressure.

---

## 5. Surviving the nightly auto-shutdown

Azure **deallocates the VM every night** to conserve credit. Two things make that a non-event:

1. **Every service declares `restart: unless-stopped`.** (The sole exception is `minio-init`, a
   one-shot bucket initialiser that is *meant* to exit 0.) Before this was fixed, only 4 of 21
   containers had a restart policy — a reboot brought the host back with the gateway, all four Spring
   services and every database **stopped**.
2. **The web UI is a systemd unit**, so it comes back too. A tmux session would not.

The public IP is **Static**, so it survives deallocation. This matters because the therapist web UI
addresses the VM by **raw IP** — a changing IP would break it every morning even though the mobile
app (which uses the domain) would be fine.

**Verified:** after an unattended reboot, all 20 containers, Caddy, DuckDNS and the web UI came back
on their own, with `https://umatter-apcs.duckdns.org/health` returning `200 healthy`.

---

## 6. 🔴 The source-of-truth rule (read this)

> **`D:\Y4-Sem 2 Thesis` on the laptop is the SINGLE SOURCE OF TRUTH for every file that is NOT
> pushed to GitHub** — `.env` files, GitHub deploy keys + `~/.ssh/config`, the **Azure VM SSH key**,
> notification's `secrets/firebase-credentials.json`, the Caddy `Caddyfile` + DuckDNS units, the
> legal pages, and the **`umatter-web.service` systemd unit**.
>
> **Whenever such a file is created/modified/deleted on the VM, immediately copy that change back to
> the matching place under `D:\Y4-Sem 2 Thesis`.**

**Why:** these files never reach GitHub. When the credit runs out the VM is gone and you may not be
able to log in to copy anything. The laptop copy is then the *only* surviving copy — it is what makes
a future rebuild possible (secrets come **from the laptop**, not from a dead VM).

The full file list is in [02-Azure-Cloud-Runbook § STEP 0](02-Azure-Cloud-Runbook.md).

---

## 7. Disaster recovery = the rebuild runbook

There is no live backup of the VM. Recovery means **rebuilding from scratch** using secrets from the
laptop: provision, push deploy keys, clone, drop `.env` + secrets, open the NSG, install Docker +
Node, create the shared network, bring up the stacks, start the web UI service, restore the HTTPS
edge, verify, repoint the clients. That entire procedure is
[02-Azure-Cloud-Runbook](02-Azure-Cloud-Runbook.md).

---

## 8. Data persistence

Postgres, MinIO, and pgAdmin use Docker named volumes, which survive reboots and `compose up`. A
**rebuild on a new VM** starts with a **fresh seed** — live data was not migrated on either the
2026-06-07 dev→prod move or the 2026-07-11 Oracle→Azure move. To carry real data forward:
`pg_dump`/restore per database + copy the MinIO buckets.
