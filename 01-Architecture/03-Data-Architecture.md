# Data Architecture

> How uMatter stores data: the database-per-service pattern, each service's schema, how cross-domain
> references are modelled without coupling, and how media is stored in MinIO.

---

## 1. Database-per-service

Every stateful service owns **exactly one PostgreSQL database**, and **no service ever reads or
writes another service's database**. This is the backbone of the microservices design.

| Service | Database | Postgres image | Host port |
|---|---|---|---|
| Auth | `auth_db` | postgres:16 | 5431 |
| AI | `ai_db` | postgres:16 | 5437 |
| Tracking | `tracking_db` | postgres:16 | 5434 |
| Therapist (Booking) | booking DB | postgres:15 | 5435 |
| Social | social DB | postgres:16 | 5436 |
| Notification | `notification` | postgres:16 | 5432 |
| Dashboard | *(none — stateless BFF)* | – | – |

**Schema management:** every service versions its schema with **Flyway** (`V1__*.sql`, `V2__*.sql`,
…) and runs Hibernate with `ddl-auto: validate`, so the running schema is always exactly what the
migrations define. Migration counts at time of writing: Auth 10, Tracking 14, AI 2, plus Therapist,
Social, and Notification (3) migration sets.

---

## 2. The golden rule: cross-domain references are plain UUIDs

When one domain needs to *point at* an entity in another domain (e.g. an appointment points at a
patient that lives in the Auth domain), it stores a **scalar UUID column** — never a JPA relationship.

```java
// CORRECT — cross-domain reference as a scalar UUID
@Column(name = "profile_id")
private UUID profileId;        // points at a Profile owned by Auth, but no FK / no join

// FORBIDDEN across services — would couple two databases
@ManyToOne @JoinColumn(name = "profile_id")
private Profile profile;       // ❌ never do this across a service boundary
```

The two most common cross-domain UUIDs system-wide:
- **`profile_id`** — a user's profile (owned by Auth). Carried by Tracking, AI, Therapist, Social,
  Notification, and inside JWTs.
- **`account_id`** — a raw account (owned by Auth). E.g. `therapists.account_id`.

Within a single service, normal FKs and `@ManyToOne` are fine.

---

## 3. Schemas by service

### 3.1 Auth Service — `auth_db`
| Table | Purpose |
|---|---|
| `users` | account credentials |
| `profiles` | app identity, role (TEEN/THERAPIST), school, avatar |
| `data_access_grants` | consent: which profile granted which other profile access |
| *(therapist profile / license tables)* | therapist directory data + license verification state |

### 3.2 Tracking Service — `tracking_db`
| Table | Purpose |
|---|---|
| `mood_logs` | mood score + emotion tags |
| `sleep_logs` | sleep duration/quality |
| `food_logs` | food intake |
| `diary_entries` | journal entries (with optional media) |
| `step_logs` | step counts |
| `breathing_logs` | breathing-exercise sessions |
| `streaks` | habit streak counters |
| `treasures` | "memory treasures" (media-rich entries) |
| `media_attachments` | references to MinIO objects |
| `data_access_grants` | Tracking-side mirror of grants (who may read this user's data) |

### 3.3 AI Service — `ai_db`
| Table | Purpose |
|---|---|
| `chat_sessions` | conversation sessions per profile |
| `chat_messages` | individual user/AI messages |

### 3.4 Therapist API — booking DB
| Table | Purpose |
|---|---|
| `therapists` | therapist directory (name, specialization, rating, matching attributes, license URL) |
| `therapist_zoom_credentials` | per-therapist static Zoom Personal Meeting Room (shared-PK via `@MapsId`) |
| `weekly_templates` | recurring weekly availability patterns |
| `schedule_slots` | concrete bookable slots (`is_booked` flag) |
| `appointments` | bookings; snapshots the Zoom room at booking time; status machine |
| `clinical_notes` | one per appointment (diagnosis, recommendations) |
| `reviews` | one per appointment; drives `therapists.rating_avg` |
| `profiles_preferences` | a patient's matching intake (orientation, reasons, communication style) |
| `therapist_assignments` | patient↔therapist pairing history (ACTIVE/INACTIVE/CHANGED_BY_REQUEST) |

**Concurrency-safe booking** uses an atomic conditional update so two patients can't grab the same slot:
```sql
UPDATE schedule_slots SET is_booked = true
WHERE slot_id = :slotId AND is_booked = false;   -- 1 row → success, 0 rows → conflict (409)
```

### 3.5 Social — social DB
| Table | Purpose |
|---|---|
| `friend_requests` / friendships | friend graph + request lifecycle |
| `blocks` | blocked profiles |
| chat `channels` | 1:1 / group chat channels |
| chat `messages` | messages, read receipts |
| `profiles` | local mirror of the minimal profile fields Social needs |

### 3.6 Notification — `notification`
| Table | Purpose |
|---|---|
| `in_app_notifications` | the inbox: title, message, `type`, `is_read`, `created_at` |
| `device_tokens` | FCM token → `profile_id` → platform registry |

`in_app_notifications.type` ∈ `BOOKING, STREAK, CHAT, REMINDER, INSIGHT` (CHECK-constrained).
Indexed by `(profile_id, created_at DESC)` to serve the inbox query directly.

---

## 4. Object storage (MinIO / S3)

Binary media (avatars, diary images, treasure photos/audio) is **not** stored in Postgres. It lives
in **MinIO**, an S3-compatible server, in the bucket **`mhsa-media`** (created by `minio-init` at
startup, versioning enabled).

- Services use the AWS S3 SDK with **path-style access** against `http://minio:9000`.
- Clients never hit MinIO directly. The service issues a **presigned GET URL** whose host is the
  **public gateway** (`S3_PUBLIC_ENDPOINT = http://<PUBLIC_IP>:8080`), and Nginx proxies
  `/mhsa-media/<key>` to MinIO, **preserving the Host header** so the SigV4 signature still validates.
- Upload limit: Nginx `client_max_body_size 30m` (treasure media up to 25 MB file / 30 MB request).

```
Mobile app ──GET http://<IP>:8080/mhsa-media/<key>?X-Amz-Signature=…──► Nginx ──► MinIO:9000
            (presigned URL minted by Tracking; Host preserved for signature validity)
```

---

## 5. Redis usage

Redis is shared infrastructure used for two distinct purposes:

1. **Caching** — services (e.g. Tracking, Auth) cache hot reads.
2. **Idempotency** — the Notification service stores `SET notif:idemp:<scope>:<messageId> "1" NX EX 86400`
   keys to ensure an at-least-once RabbitMQ event is acted on **exactly once** within 24h. See
   [04-Event-Driven-Messaging](04-Event-Driven-Messaging.md).

The "hybrid security" design also calls for Redis-backed refresh tokens (long-lived, revocable);
this is implemented in Auth but **not yet** in every service (see Therapist's known gaps).

---

## 6. Data sharing & consent (cross-service feature)

A therapist seeing a patient's tracking data is a **consent-gated, two-service** interaction:

```
1. Patient grants access:   POST /api/v1/auth/grants  { granteeProfileId: <therapist> }
                            → row in auth_db.data_access_grants
2. The grant is reflected to the Tracking domain (tracking_db.data_access_grants)
3. Therapist reads context: GET /api/v1/tracking/context/{patientId}?days=7
                            → Tracking enforces the grant before returning anything
```
No grant ⇒ no data. This keeps the privacy promise central to a mental-health product.

---

**Next:** [04-Event-Driven-Messaging](04-Event-Driven-Messaging.md) ·
[05-Security-and-Authentication](05-Security-and-Authentication.md)
