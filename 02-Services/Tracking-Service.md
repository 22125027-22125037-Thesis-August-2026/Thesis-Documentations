# Tracking Service

| | |
|---|---|
| **Port** | 8084 (host == container) |
| **Repo / module** | [`uMatter-Backend_Auth_Tracking_AI/tracking-service`](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) (Maven monorepo) |
| **Java package** | `com.mhsa.backend.tracking` |
| **Database** | `tracking_db` (PostgreSQL 16, host 5434) |
| **Tech** | Spring Boot 4.0.2, Java 17, Spring Data JPA, Flyway, Redis, RabbitMQ, MinIO/S3 |
| **Gateway prefix** | `/api/v1/tracking/` |

---

## 1. Purpose

Tracking owns all of a teen's **self-care data**: mood, sleep, food, diary, steps, breathing
exercises, habit streaks, and media-rich "treasures". It also produces the **aggregated context**
that powers the AI companion and the therapist's view of a consenting patient.

This is the richest data domain in the system (**9 Flyway migrations**, V1–V9 — the demo-data seed was
renumbered V4→V8 in July 2026 to resolve a duplicate-version clash with the grant-watermark
migration, and V9 widened `access_scope` for per-category grant sets).

---

## 2. Data model (`tracking_db`)

| Table | Purpose |
|---|---|
| `mood_logs` | mood score + emotion tags |
| `sleep_logs` | sleep duration & quality |
| `food_logs` | food intake entries |
| `diary_entries` | journal entries (optional media via multipart) |
| `step_logs` | daily step counts |
| `breathing_logs` | guided breathing sessions |
| `streaks` | habit-streak counters (gamification) |
| `treasures` | "memory treasures" — media-rich entries (image/audio in MinIO) |
| `media_attachments` | references to MinIO objects |
| `data_access_grants` | Tracking-side mirror of consent grants (who may read this user's data) |

---

## 3. API surface

All under `/api/v1/tracking/`. The log resources share a consistent CRUD shape.

| Resource | Base path | Controller | Verbs |
|---|---|---|---|
| Mood | `/moods` | `MoodLogController` | POST `/`, GET `/{profileId}`, GET `/{profileId}/{id}`, PUT `/{id}`, DELETE `/{id}` |
| Sleep | `/sleeps` | `SleepLogController` | same CRUD shape |
| Food | `/foods` | `FoodLogController` | same CRUD shape |
| Diary | `/diaries` | `DiaryEntryController` | POST `/` (multipart), GET `/{profileId}`, GET `/{profileId}/{id}`, PUT `/{id}` (multipart), DELETE `/{id}` |
| Steps | `/steps` | `StepLogController` | POST `/`, GET `/{profileId}`, GET `/{profileId}/range`, DELETE `/{id}` |
| Breathing | `/breathing` | `BreathingLogController` | POST `/`, GET `/{profileId}`, GET `/{profileId}/range`, DELETE `/{id}` |
| Streaks | `/streaks` | `StreakController` | POST `/`, GET `/`, GET `/{id}`, PUT `/{id}`, DELETE `/{id}` |
| Treasures | `/treasures` | `TreasureController` | POST `/` (multipart, up to 25 MB), GET `/`, DELETE `/{id}` |
| Media | `/media` | `MediaAttachmentController` | POST `/`, GET `/`, GET `/{id}`, PUT `/{id}`, DELETE `/{id}` |
| Grants | `/grants` | `DataAccessGrantController` | POST `/`, DELETE `/{granteeProfileId}`, GET `/` (my grants), GET `/received` |

### Internal (gateway-blocked)
- `/internal/v1/tracking/context/{profileId}` (`ContextController`) — **the aggregated context** the
  AI service consumes to ground its prompts.
- `/internal/v1/dashboard/{profileId}/summary` (`InternalController`) — summary for the Dashboard BFF.

---

## 4. Integrations

- **AI service** calls `/internal/v1/tracking/context/{profileId}` to fetch a user's recent
  tracking summary before prompting Gemini.
- **Dashboard service** calls `/internal/v1/dashboard/{profileId}/summary`.
- **Notification:** ⚠️ **not wired.** Notification consumes `streak.milestone` off `tracking.exchange`,
  but **Tracking never publishes it.** `TrackingEventPublisher` instead sends `tracking.diary.created`,
  `tracking.mood.logged`, `tracking.streak.updated`, `tracking.sleep.logged`, `tracking.food.logged`
  via the two-arg `convertAndSend(routingKey, msg)` — i.e. to the **default exchange**, landing in
  same-named queues that **no service consumes**. Dead scaffolding on both ends; see
  [04-Event-Driven-Messaging §2](../01-Architecture/04-Event-Driven-Messaging.md).
- **Inbound (dead too):** `AuthEventListener` consumes `auth.user.deleted` and `auth.user.updated`,
  but those queues are declared **unbound** in `RabbitMQConfig` and Auth's `AuthEventPublisher` only
  ever emits `auth.grant.created/revoked` and `therapist.profile.updated`. The **live** inbound path
  is `GrantEventConsumer` (§5), which *is* bound to `auth.events`.
- **MinIO:** diary/treasure/media binaries; clients receive **presigned URLs** served through the
  HTTPS edge (`S3_PUBLIC_ENDPOINT = https://umatter-apcs.duckdns.org`, Caddy → Nginx `/mhsa-media/`
  → MinIO with the Host header preserved for the SigV4 signature).
- **Consent:** enforces `data_access_grants` before returning a user's data to anyone else (§5).

---

## 5. Security & access control

- All `/api/**` endpoints require a valid JWT; the principal is the profile id (`sub`).
- Per-profile reads (`GET /{moods|sleeps|foods|diaries|steps|breathing}/{profileId}`) are guarded
  **per category** by **`AccessGuard.canReadCategory(auth, profileId, 'READ_X')`**: allowed for the
  owner themself, or for a viewer holding an **ACTIVE, unexpired grant whose scope set covers that
  category** (`AccessScopes.allows`, on the local mirror replicated from Auth). No coverage ⇒ 403.
  One mechanism for every audience — therapist, parent, friend, and the AI companion alike.
- **Scope is a set of category tokens** (`READ_MOOD`, `READ_SLEEP`, `READ_FOOD`, `READ_JOURNAL`,
  `READ_STEPS`, `READ_BREATHING`, or `READ_ALL`), so a grant can share sleep and food while
  withholding the journal — the diary endpoint requires `READ_JOURNAL` (or `READ_ALL`) specifically.
- **`/internal/v1/tracking/context/{profileId}` is consent-gated for the AI** (`ContextController`):
  it returns context only when the user has an active grant to the reserved AI-companion principal
  (`SystemProfiles.AI_COMPANION_ID`), else 403. The AI service degrades to an ungrounded reply
  (see [AI-Service §7](AI-Service.md)). *Per-category enforcement + AI gating implemented 2026-07-20.*

---

## 6. Run it

```bash
cd uMatter-Backend_Auth_Tracking_AI && docker compose up -d --build tracking-service
curl http://localhost:8084/actuator/health
```
Notable env: `SERVICE_AUTH_URL=http://auth-service:8081`, S3/MinIO keys,
`S3_PUBLIC_ENDPOINT=https://umatter-apcs.duckdns.org` (must be the public edge, since `minio:9000`
is unreachable from mobile devices).

---

## 7. Related docs
- Media/presigned-URL flow: [01-Architecture/03-Data-Architecture §4](../01-Architecture/03-Data-Architecture.md)
- In-repo controller reference: `therapist-web-ui/docs/Tracking Service API controller.md`
