# Notification Service

| | |
|---|---|
| **Port** | 8082 (host == container) |
| **Repo** | [`notification-api`](https://github.com/22125027-22125037-Thesis-August-2026/notification_api) (standalone Gradle repo) |
| **Java package** | `com.umatter.notification` |
| **Database** | `notification` (PostgreSQL 16, host 5432) |
| **Tech** | Spring Boot 3.3.5, Java 17, Spring AMQP, Data Redis, Data JPA, Mail, Thymeleaf, Actuator |
| **External** | Firebase Cloud Messaging (FCM), SMTP |
| **Gateway prefix** | `/api/v1/notification/` → service `/api/v1/` |

---

## 1. Purpose

The Notification service is a **stateful, event-driven background worker**. It:
1. **Consumes** domain events from RabbitMQ (Booking, Tracking, Social),
2. **Persists** every alert to a local **in-app inbox** (`in_app_notifications`),
3. **Dispatches** the matching outbound message via **SMTP email** or **FCM push**,
4. Exposes a small **REST API** for the client to read/update the inbox and register devices.

It is the only service that turns "something happened in another domain" into a user-facing alert.

### What it is *not*
- Not the producer of source-of-truth events. It doesn't own accounts, bookings, streaks, or chats.
- It owns exactly **two** tables: `in_app_notifications` and `device_tokens`.

---

## 2. RabbitMQ topology (consumer side)

| Domain | Exchange | Routing key | Queue | DLX / DLQ | Live? |
|---|---|---|---|---|---|
| Booking | `booking.exchange` | `appointment.booked` | `notification.booking.booked.q` | `booking.dlx` / `…booked.dlq` | ✅ **live** |
| Tracking | `tracking.exchange` | `streak.milestone` | `notification.tracking.streak.q` | `tracking.dlx` / `…streak.dlq` | ⚠️ **dormant** |
| Social | `social.exchange` | `message.missed` | `notification.social.message-missed.q` | `social.dlx` / `…message-missed.dlq` | ⚠️ **dormant** |

> ⚠️ **Two of the three consumers never receive anything.** Tracking never publishes
> `streak.milestone` and Social never publishes `message.missed` — the consumers, queues, and DLQs
> are all correctly declared, but no producer exists. Only the booking pipeline runs end-to-end.
> See [04-Event-Driven-Messaging §2](../01-Architecture/04-Event-Driven-Messaging.md).

| Routing key | Inbox `type` | Outbound |
|---|---|---|
| `appointment.booked` | `BOOKING` | **Email (HTML) + FCM push** (both) |
| `streak.milestone` | `STREAK` | FCM push |
| `message.missed` | `CHAT` | FCM push |

Listener semantics: `auto` ack, **no requeue on failure** (error handler → DLQ on first failure),
concurrency 3–10, prefetch 20. Retry/backoff is configured in `application.yml` but **not wired**
into the project's custom container factory, so it's currently dormant. Full detail:
[01-Architecture/04-Event-Driven-Messaging](../01-Architecture/04-Event-Driven-Messaging.md).

> The Notification stack **reuses the core stack's RabbitMQ** over the host gateway rather than
> running its own broker.

---

## 3. Redis idempotency

RabbitMQ is at-least-once, so each consumer dedupes before dispatch:
```
SET notif:idemp:<routingKey>:<messageId> "1" NX EX 86400   # 24h TTL
```
First-seen → proceed; duplicate → silently ack & skip. Scope = routing key (no cross-domain
collisions). Missing `messageId` → fail-open (process once, log a warning).

---

## 4. The dual-action consumer

```
1. idempotency.tryAcquire(scope, messageId)
2. historyService.save(profileId, type, title, message)     # commit inbox row (authoritative)
3. dispatch: booking → email ; tracking/social → FCM fan-out over device_tokens
```
A dead FCM token logs a warning but doesn't fail the event; a profile with zero devices still gets
the inbox row.

---

## 5. Data model (`notification`)

### `in_app_notifications`
`notification_id` (UUID PK), `profile_id`, `title`, `message`, `type` ∈
`{BOOKING, STREAK, CHAT, REMINDER, INSIGHT}` (CHECK), `is_read` (default false), `created_at`.
Indexed by `(profile_id, created_at DESC)` for the inbox query.

### `device_tokens`
`device_token` (TEXT PK), `profile_id`, `platform` ∈ `{ANDROID, IOS, WEB}` (CHECK), `created_at`,
`last_seen_at`. A profile can have many devices; re-registering a token re-binds it to a new profile
(handles "different user signs in on this device").

---

## 6. REST API — `/api/v1/notifications` & `/api/v1/devices`

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/notifications/{profileId}` | paginated inbox, newest first (`page`, `size` ≤ 100) |
| PUT | `/api/v1/notifications/{notificationId}/read` | mark one read |
| PUT | `/api/v1/notifications/{profileId}/read-all` | mark all read |
| POST | `/api/v1/devices` | register/refresh an FCM token (idempotent) |
| DELETE | `/api/v1/devices/{deviceToken}` | deregister a device (logout/invalidation) |

`GET` returns a Spring `Page<InAppNotificationDto>`. `POST /devices` body:
`{ profileId, deviceToken, platform }`.

---

## 7. Firebase & email setup

- **FCM:** uses `firebase-admin` with a service-account JSON. Save it as
  `secrets/firebase-credentials.json` (gitignored). Set `FIREBASE_CREDENTIALS_PATH` (in compose:
  `/app/secrets/firebase-credentials.json`) and `FIREBASE_ENABLED=true`. Without it, push is logged
  and skipped (`FIREBASE_ENABLED=false`) and email still works.
- **Email:** SMTP host/port/credentials + `MAIL_FROM`/`MAIL_FROM_NAME`; HTML via Thymeleaf templates.

> 🔴 The Firebase credentials file is a **source-of-truth secret** — keep a copy on the laptop per the
> rule in [05-Deployment/02-Azure-Cloud-Runbook](../05-Deployment/02-Azure-Cloud-Runbook.md).

---

## 8. Operational notes
- **Graceful shutdown:** `server.shutdown=graceful`, 30s drain; `tini` as PID-1 forwards `SIGTERM`.
- **Layering rule (one-way):** `api/consumer → service → persistence → external`. Consumers and the
  REST controller never call each other.

## 9. Run it
```bash
cd notification-api && docker compose up -d --build
curl http://localhost:8082/actuator/health
```
In-repo references: `notification-api/CONTEXT.md`, `notification-api/docs/FCM_notification.md`,
`notification-api/docs/NOTIFICATION_API_CONTROLLER_REFERENCE.md`.
