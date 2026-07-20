# Executive Summary

> **Audience:** Thesis Professor & Council. This page gives a complete, non-technical-first
> picture of what was built, why, and how it demonstrates the learning objectives of the program.

---

## 1. The problem

Adolescent mental health is a growing public-health concern, yet teenagers face three persistent
barriers to getting help: **stigma** (they avoid asking adults for help), **access** (qualified
therapists are scarce and intimidating to approach), and **continuity** (a one-off counselling
session has little context about the teen's day-to-day emotional life).

## 2. The solution — *uMatter*

**uMatter** is a mental-health support platform that addresses all three barriers in one system:

- **Lowers stigma** with private, self-directed tools: daily **mood / sleep / food / diary**
  tracking, guided **breathing** exercises, habit **streaks**, and a personal "**treasure**" memory
  journal — all on the teen's own phone.
- **Improves access** with an **AI companion** (Google Gemini) the teen can talk to any time, and a
  **therapist-matching + booking** flow that pairs a teen with a suitable, licensed therapist and
  lets them book **video consultations** without an awkward first phone call.
- **Restores continuity** by (with the teen's explicit consent) letting the matched therapist see a
  summarised view of the teen's tracking data, and by giving therapists a **clinical-notes** tool so
  care carries over between sessions.

A lightweight **social layer** (friends + real-time chat) reduces isolation, and an
**event-driven notification** system keeps everyone informed (appointment confirmations, streak
milestones, missed messages) over push and email.

## 3. What was actually built (scope delivered)

A **production-grade, cloud-deployed** distributed system, not a prototype:

| Layer | Delivered |
|---|---|
| **Backend** | **7 Spring Boot microservices** (Auth, AI, Tracking, Dashboard, Therapist/Booking, Social, Notification) behind an **Nginx API gateway**, communicating over REST and **RabbitMQ events**. |
| **Data** | **Database-per-service** (7 PostgreSQL instances), **Redis** for caching & idempotency, **MinIO** S3-compatible object storage for media, all schema-versioned with **Flyway**. |
| **Clients** | A **React Native** mobile app for teens/patients and a **React/Vite** web dashboard for therapists. |
| **AI** | A context-grounded mental-health chatbot built on **Google Gemini 2.5 Flash**, fed the user's own tracking summary. |
| **Real-time** | **STOMP-over-WebSocket** chat; **Firebase Cloud Messaging** push notifications; **Zoom SDK** video consultations. |
| **Deployment** | Fully containerised with **Docker Compose**, deployed to a **Microsoft Azure** VM as **4 stacks / 20 containers** behind an HTTPS edge, with a documented, repeatable rebuild runbook — exercised in a live Oracle→Azure migration on 2026-07-11. |

## 4. Why it is academically significant

The project is a working demonstration of **modern distributed-systems engineering**:

1. **Microservices done properly** — independent services, each owning its own database, with
   **no cross-service database coupling**; cross-domain references are carried as plain UUIDs and
   integration happens only via **APIs and events** (Domain-Driven Design bounded contexts).
2. **Event-driven architecture** — asynchronous, decoupled communication through RabbitMQ topic
   exchanges with **dead-letter queues**, **retry/backoff**, and **Redis-based idempotency** to
   guarantee at-least-once events never produce duplicate notifications.
3. **Security engineering** — stateless **JWT (RS256)** authentication with an **asymmetric key pair**
   (only Auth holds the private key; every other service verifies with the public key), role-based
   access control (Teen / Therapist / Admin), and a
   **consent-driven data-access-grant** model so a therapist only ever sees data the patient shared.
4. **Cloud-native operations** — reproducible infrastructure, an explicit port topology, a
   gateway-centric network design, and a disaster-recovery story (the documented rebuild runbook).
5. **Full-stack delivery** — two real client applications (mobile + web) consuming the same gateway.

## 5. System at a glance

```
        ┌──────────────┐         ┌──────────────────┐
        │ Mobile (RN)  │         │ Therapist Web UI │
        │  teens       │         │   (React/Vite)   │
        └──────┬───────┘         └────────┬─────────┘
               │      HTTPS/HTTP (REST + WebSocket)
               └─────────────┬─────────────┘
                             ▼
                  ┌─────────────────────┐
                  │  Nginx API Gateway  │  :8080  (single entry point)
                  └──────────┬──────────┘
        ┌──────────┬─────────┼──────────┬──────────┬───────────┐
        ▼          ▼         ▼          ▼          ▼           ▼
     Auth        AI      Tracking   Dashboard  Therapist     Social
     :8081      :8087     :8084      :8083       :8085        :8086
        │          │         │          │          │           │
        └──────────┴────┬────┴──────────┴──────────┴───────────┘
                        ▼                         ▲
                 ┌─────────────┐                  │  events
                 │  RabbitMQ   │ ────────────────►│ Notification :8082
                 │ (events)    │                  │ (push + email + inbox)
                 └─────────────┘
   Infra: PostgreSQL ×7 · Redis · MinIO (S3) · Firebase · Zoom · Gemini
```

## 6. Key numbers

- **7** microservices · **2** client apps · **6** code repositories
- **20** Docker containers across **4** Compose stacks on **1** cloud VM (Azure, 2 vCPU / 8 GiB)
- **7** independent PostgreSQL databases · **3** RabbitMQ domain exchanges
- **~80+** REST endpoints (see [04-API-Reference](../04-API-Reference/README.md))

---

**Continue reading:** [02-System-Overview](02-System-Overview.md) for the product walkthrough, or
[01-Architecture/01-System-Architecture](../01-Architecture/01-System-Architecture.md) for the
engineering deep-dive.
