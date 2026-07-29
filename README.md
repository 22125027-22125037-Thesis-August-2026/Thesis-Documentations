# uMatter — Thesis Documentation

> **Mental Health Support Application for Teenagers**
> A microservices platform pairing teen self-care (mood/sleep/diary tracking, an AI companion,
> peer social features) with professional therapy (therapist discovery, matching, booking,
> video consultations, and clinical notes).
>
> **Thesis Project — Advanced Program in Computer Science (APCS), HCMUS, August 2026 cohort.**

This folder is the **single, consolidated, source-of-truth documentation** for the entire
uMatter system: 5 backend repositories, 1 web front-end, and 1 mobile app, plus their cloud
deployment. It was assembled from the per-repository `README`, `CONTEXT`, and `docs/` files and
from a direct reading of the source code, so it reflects the **implemented reality** of the system.

---

## 👥 Who this documentation is for

This documentation is written for **three audiences**. Each has a recommended reading path:

| If you are… | Start here | Then read |
|---|---|---|
| 👨‍🏫 **A Professor / Council member** evaluating the thesis | [00-Overview/01-Executive-Summary](00-Overview/01-Executive-Summary.md) | [00-Overview/02-System-Overview](00-Overview/02-System-Overview.md) → [07-Academic](07-Academic/01-Thesis-Context-and-Future-Work.md) → [01-Architecture/01-System-Architecture](01-Architecture/01-System-Architecture.md) |
| 🎀 **Anyone who just wants to see what the app does** (demo / pitch) | [08-Product-Showcase/Mobile-App-Showcase](08-Product-Showcase/Mobile-App-Showcase.md) | [08-Product-Showcase/Therapist-Web-Showcase](08-Product-Showcase/Therapist-Web-Showcase.md) |
| 👩‍💻 **A new developer** joining the team | [06-Development/01-Developer-Onboarding](06-Development/01-Developer-Onboarding.md) | [05-Deployment/03-Local-Development-Setup](05-Deployment/03-Local-Development-Setup.md) → [02-Services](02-Services/README.md) → [04-API-Reference](04-API-Reference/README.md) |
| 🤖 **You, vibe-coding a new feature with Claude** | [06-Development/02-Coding-with-Claude](06-Development/02-Coding-with-Claude.md) | the specific [service doc](02-Services/README.md) + [01-Architecture](01-Architecture/01-System-Architecture.md) |
| 🧪 **Testing the product** (QA run, demo rehearsal, pre-defence check) | [09-Testing](09-Testing/README.md) | [09-Testing/01-Test-Environment](09-Testing/01-Test-Environment-Builds-and-Data.md) → [E2E-Mobile](09-Testing/E2E-Mobile/README.md) |

---

## 🗺️ Documentation map

```
Thesis Documentations/
├── README.md ........................... You are here (master index)
│
├── 00-Overview/ ........................ "What is this and why does it exist?"
│   ├── 01-Executive-Summary.md ......... 1-page summary for the council
│   ├── 02-System-Overview.md ........... Product, users, capabilities, big picture
│   └── 03-Glossary.md .................. Every term, acronym, and domain word
│
├── 01-Architecture/ .................... "How is it built?"
│   ├── 01-System-Architecture.md ....... Microservices, gateway, diagrams, principles
│   ├── 02-Service-Catalog-and-Ports.md . Every service, container, and port (port map)
│   ├── 03-Data-Architecture.md ......... Database-per-service, schemas, MinIO storage
│   ├── 04-Event-Driven-Messaging.md .... RabbitMQ exchanges, events, queues
│   └── 05-Security-and-Authentication.md  RS256 JWT, roles, data-access grants
│
├── 02-Services/ ........................ Deep dive per backend service
│   ├── README.md ....................... Index + responsibility matrix
│   ├── Auth-Service.md
│   ├── Tracking-Service.md
│   ├── AI-Service.md
│   ├── Dashboard-Service.md
│   ├── Therapist-API.md
│   ├── Social-API.md
│   └── Notification-API.md
│
├── 03-Frontend/ ........................ The two client applications
│   ├── Mobile-App.md ................... thesis-mobile (React Native — teens/patients)
│   └── Therapist-Web-UI.md ............. therapist-web-ui (React/Vite — therapists)
│
├── 04-API-Reference/ ................... Consolidated endpoint reference (all services)
│   └── README.md
│
├── 05-Deployment/ ...................... "How do I run / ship it?"
│   ├── 01-Deployment-Overview.md ....... Topology, Docker stacks, networks
│   ├── 02-Azure-Cloud-Runbook.md ....... Full Azure rebuild runbook + the 🔴 secrets rule
│   ├── 03-Local-Development-Setup.md .... Run the whole stack on a laptop
│   ├── 04-DNS-HTTPS-and-Play-Release.md . DuckDNS + Caddy HTTPS edge + Google Play release
│   └── 05-Oracle-Cloud-Runbook-(Retired).md  Superseded by Azure; kept for history
│
├── 06-Development/ ..................... "How do I contribute?"
│   ├── 01-Developer-Onboarding.md ...... Day-1 guide for a new team member
│   ├── 02-Coding-with-Claude.md ........ How to add features with an AI assistant
│   └── 03-Testing-and-Accounts.md ...... Test accounts, manual test flows
│
├── 07-Academic/ ........................ "Why these choices? What's next?"
│   └── 01-Thesis-Context-and-Future-Work.md
│
├── 08-Product-Showcase/ ............... "What can the app do?" (advertisement / demo tour)
│   ├── Mobile-App-Showcase.md ......... feature showcase for teens/patients (the mobile app)
│   └── Therapist-Web-Showcase.md ...... feature showcase for therapists (the web console)
│
└── 09-Testing/ ......................... "Does it actually work?" (test plans + results)
    ├── 00-Test-Strategy-and-Conventions.md  Levels, ID scheme, severity, entry/exit criteria
    ├── 01-Test-Environment-Builds-and-Data.md  Devices, builds, accounts, reset recipes
    ├── 02-Requirements-Traceability-Matrix.md  Showcase feature ↔ test plan coverage map
    └── E2E-Mobile/ ..................... Phase 1 — 15 end-to-end mobile test plans (M01…M15)
```

---

## ⚡ The 60-second summary

- **Product:** uMatter — a mental-health support app for teenagers and the therapists who help them.
- **Architecture:** 7 Spring Boot microservices behind an **Nginx API Gateway** (`:8080`), with a
  React Native mobile app for teens and a React web dashboard for therapists.
- **Patterns:** Microservices · API Gateway · Database-per-Service · Backend-for-Frontend (BFF) ·
  Event-Driven (RabbitMQ) · Stateless RS256 JWT auth with an asymmetric RSA key pair.
- **Infrastructure:** PostgreSQL (one DB per service), Redis (cache + idempotency), RabbitMQ
  (events), MinIO (S3-compatible media storage), all orchestrated with **Docker Compose**.
- **Deployment:** A single **Microsoft Azure** Ubuntu VM running **4 Docker stacks / 20 containers**,
  fronted by Nginx behind a **Caddy HTTPS edge** (`https://umatter-apcs.duckdns.org`); the therapist
  web UI is a static Vite build served by Caddy at the domain root (same origin as the API).
- **AI:** Google **Gemini 2.5 Flash** powers the in-app mental-health companion, grounded in the
  user's own tracking context.

For the canonical numbers, see [01-Architecture/02-Service-Catalog-and-Ports](01-Architecture/02-Service-Catalog-and-Ports.md).

---

## 📦 The repositories at a glance

| # | Repository (GitHub / VM dir) | Local path | Tech | Role |
|---|---|---|---|---|
| 1 | [`uMatter-Backend_Auth_Tracking_AI`](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) | `D:\Y4-Sem 2 Thesis\uMatter-Backend_Auth_Tracking_AI` | Java 17, Spring Boot 4.0, Maven (multi-module) | Auth, AI, Tracking, Dashboard services + Nginx gateway |
| 2 | [`therapist-api`](https://github.com/22125027-22125037-Thesis-August-2026/therapist-api) | `D:\Y4-Sem 2 Thesis\therapist-api` | Java 17, Spring Boot 4.0, Gradle | Therapist booking, matching, video, clinical notes |
| 3 | [`thesis_social`](https://github.com/22125027-22125037-Thesis-August-2026/thesis_social) | `D:\Y4-Sem 2 Thesis\thesis_social` | Java 17, Spring Boot 3.3, Gradle | Friends + real-time chat (STOMP/WebSocket) |
| 4 | [`notification_api`](https://github.com/22125027-22125037-Thesis-August-2026/notification_api) | `D:\Y4-Sem 2 Thesis\notification-api` | Java 17, Spring Boot 3.3, Gradle | Event-driven notifications (FCM push + email + inbox) |
| 5 | [`therapist-web-ui`](https://github.com/22125027-22125037-Thesis-August-2026/therapist-web-ui) | `D:\Y4-Sem 2 Thesis\therapist-web-ui` | React 18, TypeScript, Vite, Tailwind | Therapist web dashboard |
| 6 | [`thesis-mobile`](https://github.com/22125027-22125037-Thesis-August-2026/thesis-mobile) | `D:\Y4-Sem 2 Thesis\thesis-mobile` | React Native 0.83, React 19, TypeScript | Teen/patient mobile app |

> **Source-of-truth note:** the laptop directory `D:\Y4-Sem 2 Thesis` is the authoritative copy
> of every secret/config file that is **not** committed to GitHub (`.env` files, deploy keys, the
> Azure SSH key, Firebase credentials, the Caddy/DuckDNS units, the web-UI systemd unit). See the 🔴 rule in
> [05-Deployment/02-Azure-Cloud-Runbook](05-Deployment/02-Azure-Cloud-Runbook.md).

---

## 🧭 How to keep this documentation alive

This documentation describes the **implemented** system as of **July 2026**. When you change the
system, update the matching doc here. The most important invariants to keep accurate:

1. **The port map** — [01-Architecture/02-Service-Catalog-and-Ports](01-Architecture/02-Service-Catalog-and-Ports.md)
   is the single source of truth for ports. Each repo's `docker-compose.yml` must agree with it.
2. **The event topology** — [01-Architecture/04-Event-Driven-Messaging](01-Architecture/04-Event-Driven-Messaging.md)
   must list every exchange/routing-key/queue actually declared in code.
3. **The API surface** — [04-API-Reference](04-API-Reference/README.md) must match the controllers.
4. **The deployment facts** — [05-Deployment/01-Deployment-Overview](05-Deployment/01-Deployment-Overview.md)
   must name the VM the system actually runs on. It has moved twice.

---

*Last assembled 2026-06-16. Deployment docs rewritten 2026-07-11 for the Oracle → Azure migration.*

> **Last verified against all six code repositories and the live Azure VM: 2026-07-30.**

### Verification log

A sweep re-reads every commit since the previous date across all local *and* remote refs, greps the
docs for the affected identifiers, and folds each finding into the page that owns it. This log records
only *that* a sweep happened — deliberately not what each one found, because anything still true is
written down where it belongs and a running changelog of superseded claims is just a second place to
go stale.

| Date (ICT) | Covered | Notable outcome |
|---|---|---|
| 2026-07-20 | passes 1–3 | JWKS implemented rather than documented away (below); dormant `streak.milestone` / `message.missed` consumers deleted |
| 2026-07-29 13:11 | 5 commits | `COMPLETED` split into `PATIENT_COMPLETE` / `PROFESSIONAL_COMPLETE` / `OVERALL_COMPLETE`; `NO_SHOW` found never to exist in the backend enum |
| 2026-07-29 22:34 | 8 mobile commits + laptop↔VM reconciliation | All five deployed repos level with the laptop, `.env` files compared by SHA-256; `therapist-web-ui/.env` was the one case where the **VM** was right and the laptop stale |
| 2026-07-29 23:22 | 1 commit + a product decision | Health Connect reverted (`health.*` is restricted and would classify uMatter as a health app); video settled as **Jitsi, and only Jitsi** |
| 2026-07-30 | no new commits | Cross-document audit: seven stale claims, six older than the three preceding sweeps — public ingress, the CORS allow-list, the runbook's repoint checklist, the academic chapter's TLS limitations, and therapist-api's test state |

**The one finding that changed code, not docs.** Ten files claimed Auth published a JWKS endpoint and
that Dashboard consumed it. None of it was true, so the gap was closed instead of written down: Auth
now serves `GET /internal/v1/.well-known/jwks.json` and **all six** other services resolve verification
keys by `kid`, making a key rotation a change at Auth alone — see
[05-Security §Key distribution](01-Architecture/05-Security-and-Authentication.md). A latent bug was
fixed on the way: `BigInteger.toByteArray()`'s two's-complement sign byte made every 2048-bit modulus
serialise as 257 bytes instead of 256 (20/20 generated keys affected; Nimbus tolerates it, jose4j and
python-jose reject it). Nothing had ever called the method, so nothing had ever caught it.

> ⚠️ **Commit-following alone does not keep this accurate.** The 2026-07-30 sweep found seven stale
> claims with **zero** new commits in any repo. The reliable signal is **two documents disagreeing** —
> and the majority is not automatically right: three pages described the gateway's CORS allow-list, and
> the two that agreed were the wrong ones. Re-read
> [07-Academic §4–5](07-Academic/01-Thesis-Context-and-Future-Work.md) on every sweep; it is the page
> nobody thinks to edit when code lands, and it spent weeks telling the council the system had no TLS
> after HTTPS had shipped. Check prose under a table you just corrected, too — one fix landed in a
> header row and missed the sentence three lines below it.

**Not documented on purpose:** `thesis-mobile` carries an uncommitted `versionCode 5`. Unlanded work
stays out of the docs, so [04-DNS-HTTPS §8](05-Deployment/04-DNS-HTTPS-and-Play-Release.md) still
records `versionCode 4 / 1.1` — update it the moment that lands, as Play reserves codes permanently.
