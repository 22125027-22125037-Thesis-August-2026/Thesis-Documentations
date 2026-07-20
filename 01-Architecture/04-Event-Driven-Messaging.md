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

> **There are two distinct event-driven subsystems in uMatter**, and they look similar but exist for
> different reasons:
> 1. **User notifications (§2–§3)** — fan-out to a *person*: events become inbox rows + email/push.
>    The consumer is the **Notification** service. Idempotency is via a **Redis dedup key**.
> 2. **Cross-service data replication (§4)** — fan-out to *another service's database*: events keep a
>    local **read-model replica** in sync. Idempotency is via a **DB watermark**. This is the quieter,
>    arguably more sophisticated layer, and it's what makes database-per-service practical.

---

## 2. The exchange / routing-key / queue topology

| Producer domain | Exchange | Routing key | Consumer queue (owned by Notification) | DLX | DLQ |
|---|---|---|---|---|---|
| **Booking** (Therapist API) | `booking.exchange` | `appointment.booked` | `notification.booking.booked.q` | `booking.dlx` | `notification.booking.booked.dlq` |

**`appointment.booked` → inbox + email + push is the only notification pipeline.** That is the whole
table now, and it used to have three rows.

> **Removed July 2026: `streak.milestone` and `message.missed`.** Notification declared queues, DLQs
> and full consumers for both, but no service ever published either routing key — the producers were
> designed and never built. The queues sat bound and permanently empty while the consumer classes
> read, to anyone browsing the code, as evidence that streak and missed-message notifications
> existed. They did not. That misreading had already propagated into a manual test script telling a
> tester to exercise a path that could not work.
>
> Rather than leave live queues with no publisher, the consumers, DTOs, queue declarations and
> topology config were deleted. What the producers *do* publish, and why it goes nowhere, is
> unchanged and documented below:
> - **Tracking's** `TrackingEventPublisher` emits `tracking.streak.updated`, `tracking.mood.logged`
>   and friends straight to same-named default-exchange queues that **no service consumes**.
> - **Social's** `RabbitDomainEventPublisher` emits `social.friend_request_created`,
>   `social.friend_request_accepted` and `social.message_read` envelopes (routing key = `social`
>   prefix + `_`-separated event type) to its **own** `social.domain.events` exchange on the
>   social-stack broker, which **no service consumes** either; its `message_sent` publish is
>   commented out in `ChatService`.
>
> Reinstating either notification means building the producer first, then restoring the consumer to
> match — not the other way round.

The one consumed event maps to an inbox type and outbound channels:

| Routing key | Event DTO | Inbox `type` | Outbound channel(s) |
|---|---|---|---|
| `appointment.booked` | `BookingConfirmedEvent` | `BOOKING` | inbox row + **email (HTML)** + **FCM push** |

It saves the inbox row **first** and treats outbound delivery as best-effort: a device with no
registered token logs `push skipped`, and a per-token `NotificationDispatchException` is caught and
logged so one dead device cannot fail the event.

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
| **No requeue on failure** | `defaultRequeueRejected = false` + error handler throws `AmqpRejectAndDontRequeueException` | a failing message dead-letters to its DLQ (no infinite redelivery) |
| **Retry** | configured in `application.yml` (`enabled: true`, 3 attempts, backoff 2s → 10s) | ⚠️ **but not currently active**: the project supplies its own `SimpleRabbitListenerContainerFactory` bean without Spring Boot's `SimpleRabbitListenerContainerFactoryConfigurer`, so the `spring.rabbitmq.listener.simple.retry.*` advice chain is **not attached** to the listener. In practice a failed message **dead-letters on the first failure** rather than after 3 tries. Easy to enable by applying the configurer (or setting an advice chain). |
| **Concurrency** | 3–10 consumers/queue, prefetch 20 (set explicitly on the custom factory) | throughput under load |
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
     booking  → emailDispatcher.sendBookingConfirmation(...)   # SMTP, wrapped in try/catch
                then fanOutPush(...)                           # AND FCM push
     tracking → fanOutPush(...)                                # FCM push only
     social   → fanOutPush(...)                                # FCM push only
```
The inbox row (step 2) is the **authoritative record**; a single dead FCM token logs a warning but
does not fail the event (the user still sees the inbox row when they open the app).

---

## 4. Cross-Service Data Replication (the second event layer)

Because of **database-per-service**, no service can query another service's tables. But some services
*do* need another domain's data at read time. The solution is **event-carried state transfer**: a
service keeps a **local read-model replica** of the data it needs and keeps that replica fresh by
**consuming the owner's events**. The owning service stays the single source of truth; the replica is
*eventually consistent*.

Two more topic exchanges exist for this (beyond the notification ones in §2):

| Exchange | Owned by | Carries |
|---|---|---|
| `auth.events` | auth-service | `therapist.profile.updated`, `auth.grant.created`, `auth.grant.revoked` |
| `booking.exchange` | therapist-api | `therapist.assignment.changed` (the same exchange as §2's `appointment.booked`; consumers' `THERAPIST_API_EXCHANGE` env points here) |

### 4.1 The four replication streams

| Stream | Producer (owner) | Routing key(s) → exchange | Consumer (replica) | Queue | DLX / DLQ | Replica service |
|---|---|---|---|---|---|---|
| **Therapist profile** | auth-service | `therapist.profile.updated` → `auth.events` | therapist-api | `therapist-api.auth.therapist.profile.updated` | `…profile.dlx` / `…profile.dlq` | `TherapistProfileReplicaService` |
| **Data-access grants** | auth-service | `auth.grant.created`, `auth.grant.revoked` → `auth.events` | tracking-service | `tracking.auth.grant.created`, `tracking.auth.grant.revoked` | `tracking.auth.grant.dlx` / `…dlq` (shared) | `GrantReplicaService` |
| **Therapist↔patient assignment** | therapist-api | `therapist.assignment.changed` → `booking.exchange` | auth-service | `auth.therapist.assignment.changed` | `…changed.dlx` / `…changed.dlq` | `AssignmentReplicaService` |
| **Therapist↔patient chat link** | therapist-api | `therapist.assignment.changed` → `booking.exchange` | social-api | `social.therapist.assignment.changed` | `…changed.dlx` / `…changed.dlq` | `TherapistRelationshipService` |

What each replica is *for*:
- **Therapist profile** → therapist-api mirrors **only the auth-owned columns** (`full_name`,
  `specialization`, `about_me`, `years_experience`, `license_url`) onto its `therapists` row. It never
  touches therapist-api-owned fields (`rating_avg`, `communication_style`, `treated_challenges`,
  `is_lgbtq_allied`, `gender`, slots, Zoom) — those stay locally owned.
- **Grants** → tracking mirrors the consent grants so it can **enforce a therapist's access to patient
  data from its own DB**, with no per-request call to auth. (This is the privacy gate from
  [05-Security §5](05-Security-and-Authentication.md), made efficient.) *Tracking's `GrantEventConsumer`
  is the **only real grant consumer**. Two other listeners look like consumers but are inert:*
  - *ai-service's `RabbitMQConfig` declares four queues (`auth.user.deleted`, `auth.user.updated`,
    `auth.grant.created`, `auth.token.revoked`) with **no bindings**, and `AuthEventListener` only
    `log.info`s three of them (see [AI-Service §7](../02-Services/AI-Service.md)).*
  - *tracking-service separately declares **unbound** `auth.user.deleted` / `auth.user.updated` queues
    with an `AuthEventListener` — but Auth never publishes those routing keys at all.*

  *Because Auth publishes to the `auth.events` **topic exchange** and these queues are bound to
  nothing, they can only ever be reached via the default exchange — which nobody targets. They stay
  permanently empty.*
- **Assignment** → auth mirrors the active patient↔therapist pairing (owned by therapist-api's
  matching flow) so auth can answer "who is this patient's therapist?" locally.
- **Chat link** → social-api reacts to every newly **ACTIVE** assignment by creating the
  therapist↔patient **friendship + direct chat channel** (idempotent: an existing pair is left
  untouched). `INACTIVE` events are acked without action — a re-matched patient deliberately
  **keeps** the chat and relationship with the previous therapist. Note: social-api runs its own
  broker for chat, so this consumer opens a **second AMQP connection** to the core-stack broker
  (see §5).

### 4.2 Why these consumers differ from the Notification ones

| | Notification consumers (§2–§3) | Replication consumers (§4) |
|---|---|---|
| Effect of an event | external **side effect** (email/push) | a **DB row** the consumer owns |
| Idempotency mechanism | **Redis** `SET NX EX` on `messageId` | **DB watermark** (`last_event_at`) + write-if-changed |
| Handles out-of-order events? | no (dedup only) | **yes** (compares `occurredAt` to the watermark) |
| Ack mode / concurrency | `auto`, 3–10 consumers | **manual ack, single consumer (concurrency 1)** |

The replication consumers deliberately run **manual-ack with a single consumer** so that same-key
events stay **serialized** — which is what keeps the watermark guard correct under replays. They also
set `defaultRequeueRejected=false` with their own DLX/DLQ, exactly like the notification side.

### 4.3 Watermark idempotency (how replays & out-of-order are handled)

Each replica row stores a `last_event_at` watermark. Every consumer applies the same three rules
(see `TherapistProfileReplicaService`, `GrantReplicaService`, `AssignmentReplicaService`):

```
1. if (occurredAt < row.lastEventAt)  → DROP        # stale / out-of-order: newer state must win
2. copy fields, but only mark changed if a value actually differs   # write-if-changed
3. row.lastEventAt = occurredAt                     # advance the watermark
```

Three properties fall out of this, and they're the payoff of the design:
- **Replay-safe:** a redelivered event has `occurredAt == lastEventAt` → no change → no-op.
- **Out-of-order-safe:** a late older event (`occurredAt < lastEventAt`) is dropped, so stale data can
  never overwrite newer data. *(A Redis `SETNX` dedup key cannot do this — it only knows "seen / not
  seen", not "newer / older".)*
- **Convergent:** a converged replica writes nothing.

### 4.4 Nightly reconciliation (the self-healing safety net)

Events are the *primary* sync path, but if one is ever lost (broker outage, or a message that
dead-lettered and was never replayed), the replica would silently drift. So each replica also has a
**scheduled reconciliation job** that re-syncs from the owner's authoritative **internal REST
snapshot**. Each job is itself idempotent (a converged replica makes it a no-op). They are staggered
nightly (`Asia/Ho_Chi_Minh`):

| Replica | Reconciliation job | Schedule | Pulls snapshot from |
|---|---|---|---|
| auth — assignments | `AssignmentReconciliationService` | **23:00** | therapist-api `GET /internal/assignments` |
| tracking — grants | `GrantReconciliationService` | **23:30** | auth-service internal grants snapshot |
| therapist-api — profiles | `TherapistProfileReconciliationService` | **23:45** | auth-service `GET /internal/therapist-profiles` |

So each replica has **two redundant sync paths**: real-time events (fast, normal case) **plus** a
nightly reconcile that heals any drift (belt-and-braces). This is a pragmatic answer to the hardest
part of event-driven systems — "what if a message is lost?" — without needing a transactional outbox.

### 4.5 Replication summary diagram

```
auth-service  ─ therapist.profile.updated ─► auth.events      ─► therapist-api → TherapistProfileReplicaService → therapists (auth-owned cols)
auth-service  ─ auth.grant.created/revoked ► auth.events      ─► tracking-svc  → GrantReplicaService            → data_access_grants (replica)
therapist-api ─ therapist.assignment.changed► booking.exchange ─► auth-service  → AssignmentReplicaService       → assignment replica
therapist-api ─ therapist.assignment.changed► booking.exchange ─► social-api    → TherapistRelationshipService   → friendship + direct chat channel (ACTIVE only)

Each consumer: manual-ack · single-consumer · DLX/DLQ on failure · DB-watermark idempotency
(the social-api consumer uses exists-check idempotency instead of a watermark — it only ever adds)
Nightly reconcile (23:00 / 23:30 / 23:45 ICT) re-syncs each replica from the owner's /internal/* snapshot
```

---

## 5. Broker layout

- **Core stack RabbitMQ** (`rabbitmq:3.12-management-alpine`) on host `5671`/mgmt `15671` carries
  both layers' traffic: the notification exchanges (`booking.exchange`, `tracking.exchange`) **and**
  the replication exchanges (`auth.events`, and `booking.exchange` again for
  `therapist.assignment.changed` — §4). Auth, Tracking, and Therapist
  all connect to this broker over the `umatter-shared` network, and the Notification stack **reuses
  it via the host gateway** rather than running its own broker.
- **Social stack RabbitMQ** (`rabbitmq:3.13-management`) on host `5676`/mgmt `15676` additionally
  enables the **STOMP** (`61613`) and **Web-STOMP** (`15675`) plugins, because Social relays its
  real-time chat to WebSocket clients through RabbitMQ. See [Social-API](../02-Services/Social-API.md).

---

## 6. Producer → consumer summary diagram (notification layer)

```
Therapist API ──appointment.booked──► booking.exchange ──► notification.booking.booked.q ──► inbox + Email + FCM push

(streak.milestone and message.missed used to appear here with "(nobody)" as the producer.
 Both were removed in July 2026 — see §2.)

Therapist API (matching) also emits onto booking.exchange:
   profile.demographics.updated · tracking.mood.logged · ai.crisis.alerted

Each queue ──(on failure)──► its DLQ via x-dead-letter-exchange (retry advice is configured but
                             dormant — see §3, so a failure dead-letters on the first attempt)
Each consumer ──(Redis idempotency)──► exactly-once effect
```

For the **replication layer's** diagram, see [§4.5](#45-replication-summary-diagram).

---

## 7. Extending the event system (for new features)

**To add a new notification-triggering event:**
1. **Producer side:** publish to your domain's topic exchange with a new routing key; include a
   unique `messageId`, `occurredAt`, and the recipient's `profile_id`.
2. **Consumer side (Notification):** declare a new queue + DLX/DLQ in `RabbitConfig`, add the
   routing-key/queue strings to `application.yml` (`notification.rabbit.*`), add a DTO implementing
   `NotificationEnvelope`, write an `@RabbitListener` that follows the dual-action shape above, and
   map it to an inbox `type` and an outbound channel.
3. **Update this page** with the new row.

**To add a new cross-service replica (§4):**
1. **Producer side (the owner):** publish the change on your service's topic exchange with a routing
   key and an **`occurredAt`** timestamp (the watermark depends on it).
2. **Consumer side (the replica):** add a `…MessagingConfig` declaring a durable queue bound to the
   owner's exchange + a DLX/DLQ + a **manual-ack, single-consumer** container factory; store a
   `last_event_at` column on the replica row; write the consumer to follow the watermark rules in
   [§4.3](#43-watermark-idempotency-how-replays--out-of-order-are-handled).
3. **Add a nightly `…ReconciliationService`** that pulls the owner's `/internal/…` snapshot and
   idempotently upserts (the safety net in [§4.4](#44-nightly-reconciliation-the-self-healing-safety-net)).
4. **Update this page** ([§4.1](#41-the-four-replication-streams) table).
