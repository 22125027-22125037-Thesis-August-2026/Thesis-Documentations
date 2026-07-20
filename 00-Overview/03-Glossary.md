# Glossary

> Every acronym, domain term, and project-specific word used across this documentation.
> Sorted A–Z. Domain terms (uMatter-specific) are marked **[domain]**.

| Term | Meaning |
|---|---|
| **AMQP** | Advanced Message Queuing Protocol — the protocol RabbitMQ speaks (port 5672). |
| **Appointment** **[domain]** | A booked therapy session between a patient and a therapist, tied to one availability slot. Status flows `UPCOMING → IN_PROGRESS → COMPLETED` (or `CANCELLED`). |
| **Assignment** **[domain]** | The active pairing of a patient (`profile_id`) with a therapist, created by the matching flow. One patient has at most one `ACTIVE` assignment. |
| **BFF** | Backend-for-Frontend — a service (here, **Dashboard**) that aggregates several backend services into one client-friendly response. |
| **Bounded Context** | A DDD term: an independent domain with its own model and database. Each uMatter microservice is a bounded context. |
| **CORS** | Cross-Origin Resource Sharing — browser security rules; handled centrally by the Nginx gateway. |
| **Data-Access Grant** **[domain]** | A consent record: a patient grants a therapist permission to view their tracking data. Enforced by Auth + Tracking services. |
| **Device Token** **[domain]** | An FCM token identifying one installed app on one device; stored by the Notification service to route push notifications by `profile_id`. |
| **DLQ / DLX** | Dead-Letter Queue / Exchange — where a message goes after it fails processing (poison-message handling) in RabbitMQ. |
| **DDD** | Domain-Driven Design — the modelling approach behind the service boundaries. |
| **FCM** | Firebase Cloud Messaging — Google's push-notification service used for mobile push. |
| **Flyway** | A database-migration tool; every service versions its schema with `V1__...sql`, `V2__...sql` files. |
| **Gateway** | The **Nginx** reverse proxy at `:8080`; the single public entry point for all clients. |
| **Gemini** | Google **Gemini 2.5 Flash** — the LLM powering the AI companion. |
| **Grant** | See *Data-Access Grant*. |
| **Idempotency** **[domain]** | Guaranteeing an event processed twice has the same effect as once. The Notification service uses Redis (`SET … NX EX`) to dedupe events. |
| **Internal endpoint** **[domain]** | An endpoint under `/internal/...` for service-to-service calls only; **blocked at the gateway** so it is unreachable from the internet. |
| **JWKS** | JSON Web Key Set — the standard way to publish an RSA **public** key over HTTP so other services can verify JWTs. ⚠️ **Not implemented in uMatter:** `JwtUtils.getJwksResponse()` exists but no controller serves it, and `MHSA_APP_JWKSENDPOINT` is read by no code. Services are configured with the public key directly via `JWT_PUBLIC_KEY`. |
| **JWT** | JSON Web Token — the signed access token carried in `Authorization: Bearer …`. Signed with **RS256** (asymmetric). |
| **MHSA** | "Mental Health Support Application" — the original/internal name for the project; appears in Java package names (`com.mhsa.backend.*`) and the MinIO bucket (`mhsa-media`). Synonymous with **uMatter**. |
| **MinIO** | An S3-compatible object-storage server used for media (avatars, diary/treasure images & audio). |
| **Patient** **[domain]** | A teen user receiving care; the JWT role `TEEN` maps to Spring authority `ROLE_PATIENT`. |
| **Presigned URL** | A time-limited, signed URL that lets a client GET a private MinIO object directly (proxied via the gateway's `/mhsa-media/` route). |
| **Profile** **[domain]** | A user's public/app identity (distinct from the raw account). Most cross-service references use `profile_id` (a UUID). |
| **RabbitMQ** | The message broker carrying domain events between services. |
| **RBAC** | Role-Based Access Control — permissions derived from the JWT `role` claim (Teen/Therapist/Admin). |
| **Redis** | In-memory data store; used for caching and for notification idempotency keys. |
| **Slot** **[domain]** | A bookable window in a therapist's schedule (`schedule_slots`); booking atomically marks it `is_booked`. |
| **STOMP** | Simple Text Oriented Messaging Protocol — the messaging protocol over WebSocket used by the **Social** chat. |
| **Streak** **[domain]** | A gamified habit counter in Tracking (e.g. consecutive days of mood logging). |
| **Treasure** **[domain]** | A "memory treasure" — a media-rich journal entry (image/audio) in the Tracking service, stored in MinIO. |
| **uMatter** | The product name for the whole platform. See also *MHSA*. |
| **umatter-shared** | The **external Docker network** every Compose stack joins so containers across stacks can reach each other by name. Must be created before `compose up`. |
| **Weekly Template** **[domain]** | A therapist's recurring weekly availability pattern; a scheduled job materialises it into concrete `schedule_slots`. |
| **Zoom SDK JWT** | A short-lived token the Therapist API mints so the client Zoom SDK can join the appointment's meeting room. |

---

## Role & status enumerations (quick reference)

**JWT roles → Spring authorities**
- `TEEN` → `ROLE_PATIENT`
- `THERAPIST` → `ROLE_THERAPIST`
- `ADMIN` → `ROLE_ADMIN`

**Appointment status:** `UPCOMING` → `IN_PROGRESS` → `COMPLETED`; also `CANCELLED`.

**Assignment status:** `ACTIVE`, `INACTIVE`, `CHANGED_BY_REQUEST`.

**Notification inbox `type`:** `BOOKING`, `STREAK`, `CHAT`, `REMINDER`, `INSIGHT`.

**Device platform:** `ANDROID`, `IOS`, `WEB`.
