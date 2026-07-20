# System Overview

> **Audience:** Everyone. A product-and-architecture walkthrough that explains *what uMatter does*
> and *how the pieces fit together*, before the deep technical sections.

---

## 1. Two kinds of user, one platform

uMatter serves two primary roles, each with its own application:

| Role | App | What they do |
|---|---|---|
| **Teen / Patient** | **Mobile app** (`thesis-mobile`, React Native) | Track mood/sleep/food/diary, do breathing exercises, build streaks, keep a "treasure" memory journal, chat with the AI companion, find & book a therapist, attend video sessions, chat with friends. |
| **Therapist** | **Web dashboard** (`therapist-web-ui`, React) | Manage availability, see assigned patients, view a patient's shared tracking summary, run video consultations, write clinical notes, read reviews. |
| *(Admin)* | *(via Auth API)* | Verify/reject therapist licenses. |

Both apps talk to the **same backend**, always through the **Nginx API Gateway** at `:8080`.

---

## 2. The seven backend services and what each owns

uMatter follows **Domain-Driven Design**: the system is split into independent **bounded contexts**,
and each context is a microservice that owns its own database. No service reaches into another
service's database — they integrate only via REST calls and RabbitMQ events.

| Service | Bounded context | Owns | Headline features |
|---|---|---|---|
| **Auth Service** | Identity | `auth_db` | Registration/login, RS256 JWT issuance, profiles (Teen/Therapist), avatars, **data-access grants**, therapist license verification |
| **Tracking Service** | Self-care data | `tracking_db` | Mood, sleep, food, diary, steps, breathing logs; streaks; "treasures"; media attachments; aggregated **context** for AI & therapists |
| **AI Service** | AI companion | `ai_db` | Gemini-powered chat sessions, grounded in the user's tracking context |
| **Dashboard Service** | Aggregation (BFF) | *(stateless)* | Fans out to other services to assemble combined summaries for the clients |
| **Therapist API** | Booking | `booking` DB | Therapist directory, **matching**, availability slots, **appointment booking**, **Zoom video** join, clinical notes, reviews, scheduled slot generation |
| **Social** | Peer connection | `social` DB | Friend requests, blocking, **real-time chat** over STOMP/WebSocket |
| **Notification** | Messaging-out | `notification` DB | Consumes events from other services → **in-app inbox** + **FCM push** + **email**; device-token registry; Redis idempotency |

Full per-service detail: [02-Services](../02-Services/README.md).

---

## 3. A day in the life — three example flows

### 3.1 A teen tracks their mood and talks to the AI

```
Mobile app → Gateway → Tracking Service: POST /api/v1/tracking/moods  (saves a mood log)
Mobile app → Gateway → AI Service:       POST /api/v1/ai/chat/send
                AI Service → Tracking Service (internal): GET context summary
                AI Service → Google Gemini: prompt grounded in the teen's recent moods/sleep
                AI Service → returns an empathetic, context-aware reply
```
The AI's answer is **personalised** because the AI service pulls the teen's own recent tracking
context before calling Gemini.

### 3.2 A teen gets matched with a therapist and books a session

```
Mobile app → Gateway → Therapist API: POST /api/v1/matching/preferences  (intake form)
   • Therapist API computes compatible therapists & auto-assigns the best match
   • It also publishes integration events: profile.demographics.updated, tracking.mood.logged,
     and (if the form flags self-harm risk) ai.crisis.alerted
Mobile app → Gateway → Therapist API: GET /api/v1/therapists/{id}/slots   (available times)
Mobile app → Gateway → Therapist API: POST /api/v1/bookings              (book a slot)
   • The slot is atomically locked, an Appointment is created with a snapshot of the
     therapist's Zoom room, and an `appointment.booked` event is published
Notification Service consumes `appointment.booked` → emails a confirmation + writes an inbox row
At session time: GET /api/v1/bookings/{id}/join → returns Zoom meeting number + SDK JWT
```

### 3.3 A therapist completes a session

```
Therapist Web UI → Gateway → Therapist API: join the video call (Zoom SDK on the client)
After the call → POST /api/v1/notes (diagnosis + recommendations)
   • Appointment transitions IN_PROGRESS → COMPLETED
Later, the patient → POST /api/v1/reviews → therapist's average rating is recalculated
```

---

## 4. How the services talk to each other

There are exactly **two** integration channels, by design:

1. **Synchronous REST** — for read-time data a caller needs *right now*. These go either through the
   gateway (client → service) or directly service-to-service over the internal Docker network using
   `/internal/...` endpoints (e.g. Dashboard → Tracking, AI → Tracking, Auth → Therapist). Internal
   endpoints are **blocked at the gateway** so the public internet can never reach them.
2. **Asynchronous events** — for "something happened" facts a producer wants to announce without
   caring who listens. These flow through **RabbitMQ topic exchanges** (`booking.exchange`,
   `tracking.exchange`, `social.exchange`). The Notification service is the main consumer.

See [01-Architecture/04-Event-Driven-Messaging](../01-Architecture/04-Event-Driven-Messaging.md).

---

## 5. The supporting infrastructure

| Component | Used for | Notes |
|---|---|---|
| **PostgreSQL 15/16** | Persistent storage | One instance **per service** (database-per-service pattern) |
| **Redis** | Caching + **idempotency** | Notification uses it to deduplicate at-least-once events |
| **RabbitMQ** | Event bus | Topic exchanges, per-queue dead-letter queues, retry/backoff |
| **MinIO** | Object storage (S3 API) | Avatars, diary/treasure media; served to clients via the gateway at `/mhsa-media/` using presigned URLs |
| **Nginx** | API gateway | Single public entry point, CORS, routing, internal-endpoint blocking |
| **Firebase Cloud Messaging** | Push notifications | Notification service holds the device-token registry |
| **Zoom SDK** | Video consultations | Backend is an authorization gatekeeper; media is peer-to-peer client-side |
| **Google Gemini 2.5 Flash** | AI companion | Called by the AI service with the user's context |

---

## 6. Where everything runs

- **Production:** one **Microsoft Azure** Ubuntu 24.04 VM. The 5 backend repos run as
  **4 Docker Compose stacks** (20 containers); the therapist web UI is a **static Vite build served
  by Caddy at the domain root** (same origin as the API — the old `:5173` systemd Vite server is
  retired). Clients reach everything at **`https://umatter-apcs.duckdns.org`** (Caddy →
  Nginx gateway), so the underlying IP can change without touching any client.
- **Local development:** the same Compose stacks on a laptop; see
  [05-Deployment/03-Local-Development-Setup](../05-Deployment/03-Local-Development-Setup.md).

The complete deployment story — including the disaster-recovery rebuild runbook and the rule that
keeps secrets safe — is in [05-Deployment](../05-Deployment/01-Deployment-Overview.md).

---

**Next:** the [Glossary](03-Glossary.md) defines every term used across these docs, or jump to the
[System Architecture](../01-Architecture/01-System-Architecture.md) deep-dive.
