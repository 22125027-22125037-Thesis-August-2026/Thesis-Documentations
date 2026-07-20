# Consolidated API Reference

> Every REST endpoint across all seven services, grouped by service, derived directly from the
> controllers. Use this as the index; the in-repo `*_CONTROLLER_REFERENCE.md` files hold full
> request/response payload detail.

---

## How to read this

- **Public path** = what a client calls through the gateway at `http://<HOST>:8080…`.
- All public calls require `Authorization: Bearer <JWT>` unless marked *(public)*.
- The gateway **rewrites** some prefixes: `/api/v1/therapist/…`, `/api/v1/social/…`, and
  `/api/v1/notification/…` are proxied to each service's own `/api/v1/…`. The tables below show the
  **service-internal** path; prepend the gateway prefix when calling from a client.
- `/internal/…` and `/admin/…` endpoints are **not** reachable through the gateway (403 / not routed).

Gateway routing map: [01-Architecture/01-System-Architecture §4](../01-Architecture/01-System-Architecture.md).

---

## 1. Auth Service — `:8081` — gateway `/api/v1/auth/`, `/api/v1/patients/`

| Method | Path | Role | Purpose |
|---|---|---|---|
| POST | `/api/v1/auth/register` | *(public)* | register (teen/therapist) → tokens |
| POST | `/api/v1/auth/login` | *(public)* | login → access + refresh |
| POST | `/api/v1/auth/refresh` | *(public)* | refresh access token |
| GET | `/api/v1/auth/me` | any | current profile |
| PATCH | `/api/v1/auth/profile` | any | update profile |
| POST | `/api/v1/auth/profile/avatar` | any | upload avatar (multipart) |
| POST | `/api/v1/auth/password/change` | any | change password |
| GET | `/api/v1/auth/license` | therapist | own license status |
| POST | `/api/v1/auth/license/renew` | therapist | submit/renew license |
| POST | `/api/v1/auth/logout` | any | logout |
| POST | `/api/v1/auth/grants` | any | grant data access |
| DELETE | `/api/v1/auth/grants/{granteeProfileId}` | any | revoke grant |
| GET | `/api/v1/auth/grants/{profileId}` | any | grants given |
| GET | `/api/v1/auth/grants/{profileId}/received` | any | grants received |
| GET | `/api/v1/auth/grants/status/{otherProfileId}` | any | relationship status |
| GET | `/api/v1/patients/{profileId}` | therapist/admin | patient lookup |
| POST | `/admin/v1/therapists/{id}/license/verify` | admin | approve license |
| POST | `/admin/v1/therapists/{id}/license/reject` | admin | reject license |
| GET | `/internal/v1/.well-known/jwks.json` | *(internal)* | JWKS public key |
| GET | `/internal/v1/profile/{profileId}/summary` | *(internal)* | profile summary (BFF) |

---

## 2. Tracking Service — `:8084` — gateway `/api/v1/tracking/`

Log resources share a CRUD shape. `{profileId}` scopes reads to a user — allowed for the owner or a
viewer holding an ACTIVE unexpired grant whose **category set** covers that resource (therapist,
parent, friend, or the AI companion). E.g. the diary endpoint needs `READ_JOURNAL` (or `READ_ALL`).
The internal AI-context endpoint is grant-checked against the reserved AI-companion principal.

| Resource | Endpoints |
|---|---|
| **Mood** `/api/v1/tracking/moods` | `POST /` · `GET /{profileId}` · `GET /{profileId}/{id}` · `PUT /{id}` · `DELETE /{id}` |
| **Sleep** `/api/v1/tracking/sleeps` | same CRUD shape |
| **Food** `/api/v1/tracking/foods` | same CRUD shape |
| **Diary** `/api/v1/tracking/diaries` | `POST /` (multipart) · `GET /{profileId}` · `GET /{profileId}/{id}` · `PUT /{id}` (multipart) · `DELETE /{id}` |
| **Steps** `/api/v1/tracking/steps` | `POST /` · `GET /{profileId}` · `GET /{profileId}/range` · `DELETE /{id}` |
| **Breathing** `/api/v1/tracking/breathing` | `POST /` · `GET /{profileId}` · `GET /{profileId}/range` · `DELETE /{id}` |
| **Streaks** `/api/v1/tracking/streaks` | `POST /` · `GET /` · `GET /{id}` · `PUT /{id}` · `DELETE /{id}` |
| **Treasures** `/api/v1/tracking/treasures` | `POST /` (multipart) · `GET /` · `DELETE /{id}` |
| **Media** `/api/v1/tracking/media` | `POST /` · `GET /` · `GET /{id}` · `PUT /{id}` · `DELETE /{id}` |
| **Grants** `/api/v1/tracking/grants` | `POST /` · `DELETE /{granteeProfileId}` · `GET /` (my grants) · `GET /received` |
| *(internal)* | `GET /internal/v1/tracking/context/{profileId}` (AI grounding) · `GET /internal/v1/dashboard/{profileId}/summary` (BFF) |

---

## 3. AI Service — `:8087` — gateway `/api/v1/ai/`

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v1/ai/chat/send` | send a message → grounded Gemini reply |
| GET | `/api/v1/ai/chat/sessions` | list sessions |
| GET | `/api/v1/ai/chat/history/{sessionId}` | session message history |
| GET | `/internal/v1/dashboard/{profileId}/chat-stats` | *(internal)* chat stats (BFF) |

---

## 4. Dashboard Service (BFF) — `:8083` — gateway `/api/v1/dashboard/`

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/dashboard/summary` | aggregated summary for the user |
| GET | `/api/v1/dashboard/context/{profileId}` | composed context |
| GET | `/api/v1/dashboard/health` | health |

---

## 5. Therapist API — `:8085` — gateway `/api/v1/therapist/` → `/api/v1/`

| Area | Method | Path | Role |
|---|---|---|---|
| Appointments | POST | `/api/v1/bookings` | patient |
| | GET | `/api/v1/bookings/{appointmentId}` | self |
| | GET | `/api/v1/bookings/{appointmentId}/join` | self (Zoom credentials) |
| | POST | `/api/v1/bookings/{appointmentId}/cancel` | self |
| | POST | `/api/v1/bookings/{appointmentId}/confirm` | therapist |
| | POST | `/api/v1/bookings/{appointmentId}/reject` | therapist |
| | GET | `/api/v1/profiles/{profileId}/appointments/upcoming` | self |
| | GET | `/api/v1/profiles/{profileId}/appointments/history` | self |
| | GET | `/api/v1/profiles/{profileId}/appointments/unreviewed` | self |
| | GET | `/api/v1/therapists/{therapistId}/appointments` | therapist |
| Availability | GET | `/api/v1/therapists/{therapistId}/slots/manage` | therapist |
| | POST | `/api/v1/therapists/{therapistId}/slots` | therapist |
| | POST | `/api/v1/therapists/{therapistId}/slots:bulk` | therapist |
| | PUT/DELETE | `/api/v1/therapists/{therapistId}/slots/{slotId}` | therapist |
| | GET/POST | `/api/v1/therapists/{therapistId}/availability-templates` | therapist |
| | PUT/DELETE | `/api/v1/therapists/{therapistId}/availability-templates/{templateId}` | therapist |
| Therapists | GET | `/api/v1/therapists/{id}` | any |
| | GET | `/api/v1/therapists/{id}/slots` | any (paginated) |
| | GET | `/api/v1/therapists/{id}/reviews` | any |
| Matching | POST | `/api/v1/matching/preferences` | patient |
| | GET | `/api/v1/matching/therapists` | patient |
| | POST | `/api/v1/matching/assign/{therapistId}` | patient |
| Assignment | GET | `/api/v1/profiles/{profileId}/assigned-therapist` | self-or-admin |
| Clinical notes | GET | `/api/v1/notes/appointments/{appointmentId}` | therapist |
| | GET/PUT | `/api/v1/notes/{noteId}` | therapist |
| | POST | `/api/v1/notes` | therapist (create) |
| | POST | `/api/v1/notes/{noteId}/finalize` | therapist |
| Reviews | POST | `/api/v1/reviews` | patient |
| Dashboard | GET | `/api/v1/therapists/{therapistId}/dashboard/summary` | therapist |
| Patients | GET | `/api/v1/therapists/{therapistId}/patients` | therapist |
| | GET/PUT | `/api/v1/patients/{profileId}/tags` | therapist |
| | GET/PUT | `/api/v1/patients/{profileId}/risk-level` | therapist |
| | GET | `/api/v1/patients/{profileId}/matching-preferences` | therapist |
| *(internal)* | GET | `/internal/assignments` | internal |
| *(test)* | POST | `/api/v1/test/trigger-generation`, `/api/v1/test/trigger-cleanup` | permit-all |

---

## 6. Social Service — `:8086` — gateway `/api/v1/social/` → `/api/v1/`

| Area | Method | Path |
|---|---|---|
| Friends | POST | `/api/v1/friends/requests` |
| | DELETE | `/api/v1/friends/requests/{requestId}` |
| | POST | `/api/v1/friends/requests/{requestId}/accept` |
| | POST | `/api/v1/friends/requests/{requestId}/reject` |
| | GET | `/api/v1/friends/requests/incoming` · `/outgoing` · `/requests` |
| | DELETE | `/api/v1/friends/{profileId}` |
| | POST/DELETE | `/api/v1/friends/blocks/{profileId}` |
| Chat (REST) | POST | `/api/v1/chats/channels` |
| | GET | `/api/v1/chats/channels` |
| | GET | `/api/v1/chats/channels/{channelId}/messages` |
| | PATCH | `/api/v1/chats/channels/{channelId}/messages/{messageId}/read` |
| Chat (live) | WS/STOMP | WebSocket to `:8086` (`ChatWebSocketController`) |

---

## 7. Notification Service — `:8082` — gateway `/api/v1/notification/` → `/api/v1/`

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/notifications/{profileId}` | paginated inbox (`page`, `size` ≤ 100) |
| PUT | `/api/v1/notifications/{notificationId}/read` | mark one read |
| PUT | `/api/v1/notifications/{profileId}/read-all` | mark all read |
| POST | `/api/v1/devices` | register/refresh FCM device token |
| DELETE | `/api/v1/devices/{deviceToken}` | deregister device |

---

## Error model (Therapist API, representative)

Services return RFC 7807 `ProblemDetail`:

| Exception | HTTP |
|---|---|
| `SlotAlreadyBookedException`, `InvalidAppointmentStateException`, `*AlreadyExistsException` | 409 |
| `ResourceNotFoundException` | 404 |
| `MeetingNotOpenException`, `ReviewNotAllowedException` | 403 |
| validation failure | 400 (with `errors` map) |
| uncaught | 500 |

---

## In-repo full references (payload-level detail)
- `therapist-web-ui/docs/THERAPIST_API_CONTROLLER_REFERENCE.md`
- `therapist-web-ui/docs/Authentication Service API controller.md`
- `therapist-web-ui/docs/Tracking Service API controller.md`
- `thesis_social/docs/SOCIAL_API_CONTROLLER_REFERENCE.md`
- `notification-api/docs/NOTIFICATION_API_CONTROLLER_REFERENCE.md`
