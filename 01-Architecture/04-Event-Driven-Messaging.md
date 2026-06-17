# Event-Driven Messaging (RabbitMQ)

> How uMatter services announce facts to each other asynchronously. This page is the canonical
> map of every exchange, routing key, queue, and consumer in the system.

---

## 1. Why events

Some interactions must **not** be synchronous. When a patient books an appointment, the Booking
service shouldn't block on the Notification service being healthy just to send a confirmation email.
Instead it **publishes a fact** ("an appointment was booked") and moves on. Whoever cares subscribes.

The contract:
- **Producers** commit their local transaction, *then* publish. They never depend on a consumer
  being up, and they never know who (if anyone) is listening.
- **Consumers** own their queues, are idempotent, and dead-letter poison messages instead of
  looping on them.

All exchanges are **topic** exchanges; each domain owns its own exchange.

---

## 2. The exchange / routing-key / queue topology

| Producer domain | Exchange | Routing key | Consumer queue (owned by Notification) | DLX | DLQ |
|---|---|---|---|---|---|
| **Booking** (Therapist API) | `booking.exchange` | `appointment.booked` | `notification.booking.booked.q` | `booking.dlx` | `notification.booking.booked.dlq` |
| **Tracking** | `tracking.exchange` | `streak.milestone` | `notification.tracking.streak.q` | `tracking.dlx` | `notification.tracking.streak.dlq` |
| **Social** | `social.exchange` | `message.missed` | `notification.social.message-missed.q` | `social.dlx` | `notification.social.message-missed.dlq` |

Each consumed event maps to an inbox type and an outbound channel:

| Routing key | Event DTO | Inbox `type` | Outbound channel |
|---|---|---|---|
| `appointment.booked` | `BookingConfirmedEvent` | `BOOKING` | **Email** (HTML) |
| `streak.milestone` | `StreakMilestoneEvent` | `STREAK` | **FCM push** |
| `message.missed` | `MessageMissedEvent` | `CHAT` | **FCM push** |

### Additional integration events (matching workflow)
The Therapist API's matching flow publishes several **cross-domain integration events** onto
`booking.exchange` for other domains to consume:

| Routing key | Carried when | Payload |
|---|---|---|
| `profile.demographics.updated` | a patient submits the matching intake form | demographics |
| `tracking.mood.logged` | same | mood levels + timestamp |
| `ai.crisis.alerted` | the intake flags self-harm risk ("Có"/"Yes") | `source = INTAKE_FORM` |

> **Envelope contract:** every event payload implements `NotificationEnvelope` and **must** carry a
> unique `messageId` (for idempotency) and `occurredAt`. Producers carry **`profile_id`**, never
> device tokens — token→device routing is the Notification service's job.

---

## 3. Reliability mechanisms

The Notification consumers are engineered for **at-least-once** delivery without user-visible
duplicates or poison-message loops:

| Mechanism | Setting | Effect |
|---|---|---|
| **Acknowledgement** | `auto` | Spring acks on success, nacks on thrown exception |
| **No requeue on failure** | `defaultRequeueRejected = false` | a failing message goes straight to its DLQ (no infinite redelivery) |
| **Retry** | 3 attempts, exponential backoff 2s → 10s | transient failures recover before dead-lettering |
| **Concurrency** | 3–10 consumers/queue, prefetch 20 | throughput under load |
| **Idempotency** | Redis `SET notif:idemp:<scope>:<messageId> "1" NX EX 86400` | duplicate `messageId` is silently acked & skipped |

### The idempotency check, precisely
Before dispatching, each consumer calls `IdempotencyService.tryAcquire(scope, messageId)`:
- **First time:** Redis `SETNX` succeeds (key set, 24h TTL) → proceed.
- **Duplicate:** `SETNX` fails (key exists) → silently ack and skip.
- The **scope** is the routing key, so the same `messageId` can't collide across domains.
- **Fail-open:** if `messageId` is missing, log a warning and process once (the producer is the bug;
  dropping the event would be worse than a possible duplicate).

### The dual-action consumer
Every consumer does **both** a DB write and a dispatch, in this order:
```
1. idempotency.tryAcquire(routingKey, messageId)   # Redis — short-circuit duplicates
2. historyService.save(profileId, type, title, …)  # Postgres — commit the inbox row
3. dispatch:
     booking  → emailDispatcher.send(...)          # SMTP
     tracking → for each device_token: FCM push
     social   → for each device_token: FCM push
```
The inbox row (step 2) is the **authoritative record**; a single dead FCM token logs a warning but
does not fail the event (the user still sees the inbox row when they open the app).

---

## 4. Broker layout

- **Core stack RabbitMQ** (`rabbitmq:3.12-management-alpine`) on host `5671`/mgmt `15671` carries
  the `booking.exchange` and `tracking.exchange` traffic. The Notification stack **reuses Auth's
  RabbitMQ via the host gateway** rather than running its own broker.
- **Social stack RabbitMQ** (`rabbitmq:3.13-management`) on host `5676`/mgmt `15676` additionally
  enables the **STOMP** (`61613`) and **Web-STOMP** (`15675`) plugins, because Social relays its
  real-time chat to WebSocket clients through RabbitMQ. See [Social-API](../02-Services/Social-API.md).

---

## 5. Producer → consumer summary diagram

```
Therapist API ──appointment.booked──► booking.exchange ──► notification.booking.booked.q ──► Email
Tracking      ──streak.milestone────► tracking.exchange ─► notification.tracking.streak.q ─► FCM push
Social        ──message.missed──────► social.exchange ───► notification.social.…q ─────────► FCM push

Therapist API (matching) also emits onto booking.exchange:
   profile.demographics.updated · tracking.mood.logged · ai.crisis.alerted

Each queue ──(on failure: retry ×3 → )──► its DLQ via x-dead-letter-exchange
Each consumer ──(Redis idempotency)──► exactly-once effect
```

---

## 6. Extending the event system (for new features)

To add a new notification-triggering event:
1. **Producer side:** publish to your domain's topic exchange with a new routing key; include a
   unique `messageId`, `occurredAt`, and the recipient's `profile_id`.
2. **Consumer side (Notification):** declare a new queue + DLX/DLQ in `RabbitConfig`, add the
   routing-key/queue strings to `application.yml` (`notification.rabbit.*`), add a DTO implementing
   `NotificationEnvelope`, write an `@RabbitListener` that follows the dual-action shape above, and
   map it to an inbox `type` and an outbound channel.
3. **Update this page** with the new row.
