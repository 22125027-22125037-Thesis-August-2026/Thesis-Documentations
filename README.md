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
│   └── 05-Security-and-Authentication.md  JWT/JWKS, roles, data-access grants
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
│   ├── 02-Oracle-Cloud-Runbook.md ...... Full OCI rebuild runbook + the 🔴 secrets rule
│   └── 03-Local-Development-Setup.md .... Run the whole stack on a laptop
│
├── 06-Development/ ..................... "How do I contribute?"
│   ├── 01-Developer-Onboarding.md ...... Day-1 guide for a new team member
│   ├── 02-Coding-with-Claude.md ........ How to add features with an AI assistant
│   └── 03-Testing-and-Accounts.md ...... Test accounts, manual test flows
│
├── 07-Academic/ ........................ "Why these choices? What's next?"
│   └── 01-Thesis-Context-and-Future-Work.md
│
└── 08-Product-Showcase/ ............... "What can the app do?" (advertisement / demo tour)
    ├── Mobile-App-Showcase.md ......... feature showcase for teens/patients (the mobile app)
    └── Therapist-Web-Showcase.md ...... feature showcase for therapists (the web console)
```

---

## ⚡ The 60-second summary

- **Product:** uMatter — a mental-health support app for teenagers and the therapists who help them.
- **Architecture:** 7 Spring Boot microservices behind an **Nginx API Gateway** (`:8080`), with a
  React Native mobile app for teens and a React web dashboard for therapists.
- **Patterns:** Microservices · API Gateway · Database-per-Service · Backend-for-Frontend (BFF) ·
  Event-Driven (RabbitMQ) · Stateless JWT auth with a JWKS-published RSA key.
- **Infrastructure:** PostgreSQL (one DB per service), Redis (cache + idempotency), RabbitMQ
  (events), MinIO (S3-compatible media storage), all orchestrated with **Docker Compose**.
- **Deployment:** A single **Oracle Cloud (OCI)** Ubuntu VM running **4 Docker stacks / ~24
  containers**, fronted by Nginx; the therapist web UI runs as a Vite dev server alongside.
- **AI:** Google **Gemini 2.5 Flash** powers the in-app mental-health companion, grounded in the
  user's own tracking context.

For the canonical numbers, see [01-Architecture/02-Service-Catalog-and-Ports](01-Architecture/02-Service-Catalog-and-Ports.md).

---

## 📦 The repositories at a glance

| # | Repository (GitHub / VM dir) | Local path | Tech | Role |
|---|---|---|---|---|
| 1 | `uMatter-Backend_Auth_Tracking_AI` | `D:\Y4-Sem 2 Thesis\uMatter-Backend_Auth_Tracking_AI` | Java 17, Spring Boot 4.0, Maven (multi-module) | Auth, AI, Tracking, Dashboard services + Nginx gateway |
| 2 | `therapist-api` | `D:\Y4-Sem 2 Thesis\therapist-api` | Java 17, Spring Boot 4.0, Gradle | Therapist booking, matching, video, clinical notes |
| 3 | `thesis_social` | `D:\Y4-Sem 2 Thesis\thesis_social` | Java 17, Spring Boot 3.3, Gradle | Friends + real-time chat (STOMP/WebSocket) |
| 4 | `notification_api` | `D:\Y4-Sem 2 Thesis\notification-api` | Java 17, Spring Boot 3.3, Gradle | Event-driven notifications (FCM push + email + inbox) |
| 5 | `therapist-web-ui` | `D:\Y4-Sem 2 Thesis\therapist-web-ui` | React 18, TypeScript, Vite, Tailwind | Therapist web dashboard |
| 6 | `thesis-mobile` | `D:\Y4-Sem 2 Thesis\thesis-mobile` | React Native 0.83, React 19, TypeScript | Teen/patient mobile app |

> **Source-of-truth note:** the laptop directory `D:\Y4-Sem 2 Thesis` is the authoritative copy
> of every secret/config file that is **not** committed to GitHub (`.env` files, deploy keys, the
> Oracle SSH key, Firebase credentials). See the 🔴 rule in
> [05-Deployment/02-Oracle-Cloud-Runbook](05-Deployment/02-Oracle-Cloud-Runbook.md).

---

## 🧭 How to keep this documentation alive

This documentation describes the **implemented** system as of **June 2026**. When you change the
system, update the matching doc here. The most important invariants to keep accurate:

1. **The port map** — [01-Architecture/02-Service-Catalog-and-Ports](01-Architecture/02-Service-Catalog-and-Ports.md)
   is the single source of truth for ports. Each repo's `docker-compose.yml` must agree with it.
2. **The event topology** — [01-Architecture/04-Event-Driven-Messaging](01-Architecture/04-Event-Driven-Messaging.md)
   must list every exchange/routing-key/queue actually declared in code.
3. **The API surface** — [04-API-Reference](04-API-Reference/README.md) must match the controllers.

*Last assembled: 2026-06-16.*
