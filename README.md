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

*Last assembled: 2026-06-16. Deployment docs rewritten 2026-07-11 for the Oracle → Azure migration.*

> **Last verified against all six code repositories: 2026-07-29 22:34 ICT.** See the fourth- and
> fifth-pass notes at the end of this section for what changed since the previous checkpoint
> (2026-07-20 19:24 ICT).

*Verified against source code 2026-07-20 (first pass): V6 users→profiles merge, HTTPS/same-origin web
UI, grant model details (scope/expiry, enforcement caveats, the AI-service grant bypass), and the real
event topology (two dormant notification flows) folded in.*

*Second verification pass, 2026-07-20 — corrections made:*
- **JWKS was documented but not implemented — so it was implemented.** Ten files claimed Auth
  publishes a JWKS endpoint and that Dashboard fetches from it. None of it was true:
  `JwtUtils.getJwksResponse()` was called by no controller, `MHSA_APP_JWKSENDPOINT` was bound by no
  `@Value`, and Dashboard verified with a static `JWT_PUBLIC_KEY` like everyone else. Rather than
  document the gap, the gap was closed — see the code-change note below.
- **Two service docs contradicted the event map.** Tracking-Service claimed it "publishes
  streak-milestone events that become notifications" and Social-API claimed it "produces
  `message.missed`". Neither is true — both now match [04-Event-Driven-Messaging](01-Architecture/04-Event-Driven-Messaging.md),
  and the affected manual test flow in [06-Development/03](06-Development/03-Testing-and-Accounts.md) is fixed.
  The dormant consumers behind those claims have since been **deleted** (below).
- **`appointment.booked` sends email *and* push**, not email alone; and it fires **at booking time**,
  not when the therapist confirms (the showcase said otherwise).
- **API reference gaps:** added `GET /internal/grants`, `GET /internal/therapist-profiles`,
  `GET /api/v1/notes`, `GET /api/v1/friends`; removed the phantom JWKS row.
- **Container names:** therapist-api's compose sets no `container_name`, so its containers are
  `therapist-api-api-1` / `therapist-api-postgres-1` (the port map claimed otherwise; the runbooks
  were already right). `therapist-api` is a *network alias*, not a container name.
- Also fixed: tracking migration count (8 → 9), "three replication streams" (there are four),
  Social's real routing keys (`social.message_read`, not `social.message.read`), the mobile app's
  transport (HTTPS/WSS in **all** builds, no raw IP anywhere in `src/`), and a note that
  `therapist-web-ui/.env` holds five dead vars with wrong ports.

*Code changed to match the docs, 2026-07-20* — two findings were fixed in source rather than written
down as gaps:

- **JWKS now works end to end.** Auth serves `GET /internal/v1/.well-known/jwks.json`
  (`JwksController`); Dashboard resolves verification keys from it by `kid` (`JwksKeyProvider` in
  `shared-jwt`), fetching lazily so Auth is not a boot dependency, caching by kid, and refetching at
  most once a minute when an unknown kid appears. Dashboard is now handed **no signing material at
  all** — it previously received the RSA *private* key, which it never used and had no business
  holding.
  - **A latent bug was fixed on the way.** `getJwksResponse()` encoded the modulus with
    `BigInteger.toByteArray()`, whose two's-complement sign byte makes a 2048-bit modulus serialise
    as 257 bytes instead of 256. Measured across 20 generated keys: **20/20 were affected**. Nimbus
    tolerates it, jose4j and python-jose reject it. Nothing had ever called the method, so nothing
    had ever caught it.
  - **Rotation no longer implies re-login.** Setting `mhsa.app.jwtPreviousPublicKey` +
    `…PreviousSigningKid` publishes both keys during an overlap, so tokens already issued keep
    validating until they expire.

*Third pass, 2026-07-20 — the remaining five verifiers were migrated, so JWKS is now system-wide:*
- **All six non-Auth services resolve keys from the key set.** Only Auth holds signing material, and
  rotating the pair is now a change at Auth alone. Three client implementations were required,
  because the services do not share a JWT stack: `JwksKeyProvider` (`shared-jwt`) for Tracking and
  AI; `NimbusJwtDecoder.withJwkSetUri` for Notification and Social, both of which already used
  Nimbus; and a new `JwksVerificationKeyLocator` on jjwt's `Locator` SPI for Therapist, which is a
  standalone repo with no `shared-jwt` dependency and parses with jjwt.
- **Tracking and AI were also handed the RSA *private* key and never used it.** Same defect as
  Dashboard, and — as there — removing it from compose was not enough: each `entrypoint.sh`
  re-derives it from `JWT_PRIVATE_KEY`, which arrives via `env_file: .env`. Fixed in the entrypoints
  and **verified by reading `/proc/1/cmdline` in the running containers**, not by trusting the diff.
- **A pinned `kid` would have broken rotation in two services.** Therapist and Social both enforced
  an expected-`kid`; under JWKS that rejects the first token signed with a new key. Both now skip the
  pin when a JWKS URI is set.
- **Proven end-to-end on prod with a real signed token** (a throwaway account, since the seeded
  accounts cannot log in): Dashboard, Tracking, AI, Social and Notification all returned **200**;
  Therapist returned **403** on a therapist-only endpoint called by a teen — authenticated, then
  authorization-denied — while a garbage signature returned **401** everywhere. The therapist log
  shows the lazy JWKS fetch firing on that first token, eight minutes after startup.
- **Also found while testing** (documented in [06-Development/03](06-Development/03-Testing-and-Accounts.md),
  not caused by this work): therapist-api's **test suite does not compile** on `main`, so none of its
  advertised coverage runs; social has 2 deterministic failures tied to the commented-out
  `message_sent` event plus a ~50% Mockito flake; and the monorepo's `./mvnw` is broken (missing
  `.mvn/wrapper/maven-wrapper.properties`).
- **The dormant notification consumers were deleted.** `streak.milestone` and `message.missed` had
  queues, DLQs, DTOs and consumers in `notification-api`, and no producer anywhere. Live queues with
  no publisher are what caused a service doc and a manual test script to describe features that did
  not exist, so the consumer side was removed rather than left as bait.

*Fourth pass — **sync checkpoint 2026-07-29 13:11 ICT**, covering every commit in all six code repos
since the previous checkpoint of **2026-07-20 19:24 ICT**.* Five commits landed in that window, all of
them one change and its client follow-ups; `uMatter-Backend_Auth_Tracking_AI`, `thesis_social` and
`notification-api` had **zero** commits. Method: `git fetch --all` on each repo, then
`git log --all --since=…` across every local *and* remote ref (all six are level with `origin/main`,
no unmerged branches, no post-checkpoint stashes).

- **`COMPLETED` was split three ways** (therapist-api `1c1a3cc`, Flyway `V11`): `PATIENT_COMPLETE`
  (reviewed, no note), `PROFESSIONAL_COMPLETE` (note, no review), `OVERALL_COMPLETE` (both). A review
  and a clinical note used to *each* flip an appointment straight to `COMPLETED`, so the status could
  not tell you whether the note existed. The therapist also gained a **24-hour grace window** to file
  a note after a patient-first review. Folded into [Therapist-API §3](02-Services/Therapist-API.md)
  (new status-machine section), the [glossary](00-Overview/03-Glossary.md), the
  [system overview](00-Overview/02-System-Overview.md) flow, both
  [front-end docs](03-Frontend/Mobile-App.md), the [manual test flows](06-Development/03-Testing-and-Accounts.md),
  and [E2E-M08](09-Testing/E2E-Mobile/E2E-M08-Therapist-Matching-and-Booking.md) / [M09](09-Testing/E2E-Mobile/E2E-M09-Consultation-Session-and-Aftercare.md).
- **`NO_SHOW` never existed in the backend enum** (therapist-web-ui `cdf99f0`). Spring fails the
  *whole* multi-value `@RequestParam` conversion on one bad token, so the web UI's "Past" tab was
  `500`ing and silently hiding **every** completed appointment. Documented as a standing warning in
  the service doc, the glossary and M08 — this is a trap any client can fall into again.
- **E2E-M09 was documenting behaviour the system does not have.** M09-06 claimed ending a call moves
  the appointment out of `IN_PROGRESS` and treated a stuck `IN_PROGRESS` as a defect. Nothing does
  that — there is no scheduled job, and status advances *only* on a finalized note or a review. A
  tester following the old script would have filed a false bug. Inverted: `IN_PROGRESS` after hangup
  is now the expected result. M09-10 likewise claimed reviews require a completed session; the backend
  only rejects `UPCOMING`/`CANCELLED` plus a 1-minute-after-start rule.
- **Known drift, deliberately not fixed here:** the docs contradict themselves on the video provider —
  [Therapist-API](02-Services/Therapist-API.md) and [Mobile-App](03-Frontend/Mobile-App.md) describe
  Zoom (meeting number + SDK JWT), while [E2E-M09](09-Testing/E2E-Mobile/E2E-M09-Consultation-Session-and-Aftercare.md)
  describes a Jitsi WebView on `meet.jit.si`. The code supports both readings: therapist-api defaults
  `VIDEO_PROVIDER=zoom`, but `VideoConsultationScreen.tsx` is entirely Jitsi. **This predates the
  2026-07-20 checkpoint** and needs a decision, not a doc edit.
- **`thesis-mobile` carries uncommitted work** (chat screens, `versionCode 4`, and an `axiosClient.ts`
  change pointing dev builds at HTTPS instead of `http://85.211.241.204:8080`). Not documented on
  purpose — it has not landed, and the last item will contradict
  [Mobile-App.md](03-Frontend/Mobile-App.md) once it does.

*Fifth pass — **sync checkpoint 2026-07-29 22:34 ICT**, covering 8 commits pushed to `thesis-mobile`
that evening (22:07–22:22).* The other five repos were unchanged since the fourth pass. Most of this
is the work flagged as "uncommitted" above, now landed — plus one genuinely new feature.

- **Health Connect step back-fill** (`a903a11`) — the one feature. The hardware sensor reports a single
  cumulative since-boot value, so **a day the user never opened the app was lost** and nothing could
  recover it. Two Kotlin native modules now exist (`StepCounterModule` + `HealthConnectModule`), and
  `syncRecentDays()` reconciles the trailing 7 days: today from the live sensor, prior days from
  Health Connect. Documented in [Mobile-App.md](03-Frontend/Mobile-App.md) and, in detail, in
  [E2E-M05](09-Testing/E2E-Mobile/E2E-M05-Automatic-Step-Counting.md) — the never-regress rule, the
  ask-once permission, the new `HEALTH_CONNECT` source tag, and the fact that a device **without**
  Health Connect behaves exactly as before (Blocked, not Failed).
  - The new `source` value needed **no backend change**: `step_logs.source` is a free
    `VARCHAR(50) NOT NULL DEFAULT 'DEVICE_SENSOR'` with no CHECK constraint. Verified, because this is
    the same shape as the `NO_SHOW` bug and was worth ruling out rather than assuming.
- **🔴 It also created a Play release blocker.** `android.permission.health.*` is a **restricted**
  permission — Play rejects the upload unless the Health apps declaration form is approved first.
  Recorded in [04-DNS-HTTPS-and-Play-Release](05-Deployment/04-DNS-HTTPS-and-Play-Release.md) §8.
- **Dev builds now use HTTPS** (`43c3d76`), so `axiosClient.ts`'s `__DEV__` ternary has two identical
  branches. [05-Security §6](01-Architecture/05-Security-and-Authentication.md) already described it
  this way and is now correct; [Mobile-App.md](03-Frontend/Mobile-App.md) still claimed dev builds hit
  `http://<PUBLIC_IP>:8080` and was fixed. The two docs had contradicted each other.
- **Release identity was stale in two ways** (`7c46759`, `3eeccb4`): the Play section said
  `com.thesisapp` / versionCode 1, when the real id is `com.apcsthesisteam.umatter` and the next upload
  must be **versionCode 4 / 1.1** (Play reserves codes permanently). Debug builds now install
  side-by-side via an `.dev` suffix — added to [09-Testing/01](09-Testing/01-Test-Environment-Builds-and-Data.md)
  because opening the wrong icon is now a real way to invalidate a test run.
- **Mood-only diary entries used to fail to save** (`2005bd0`) — the UI treats title/tags/note as
  optional but the API rejects blank content. Noted in [E2E-M02](09-Testing/E2E-Mobile/E2E-M02-Emotional-Journal.md),
  since the fallback makes `content` read as the emotion label rather than being empty.
- *No doc impact:* `a4a49b0` (chat composer above the Android 16 IME — no doc states keyboard
  behaviour or `targetSdk`) and `b53a361` (lockfile marker refresh, no version changes).

*Confirmed accurate and left unchanged:* the full port map and 20-container count, all four compose
stacks, Spring Boot/Java/React/RN versions, Gemini 2.5 Flash, the gateway route table, the grant model
(per-category `AccessScopes` enforcement + the AI companion as a seeded grantee), the inert
token-blacklist analysis, the dormant-retry analysis in the Notification consumers, the watermark
replication design, and the nightly reconciliation schedule.*
