# Therapist API (Booking Domain)

| | |
|---|---|
| **Port** | 8085 (host == container) |
| **Repo** | [`therapist-api`](https://github.com/22125027-22125037-Thesis-August-2026/therapist-api) (standalone Gradle repo) |
| **Java package** | `com.booking.therapist_api` |
| **Database** | booking DB (PostgreSQL 15, host 5435) |
| **Tech** | Spring Boot 4.0.5, Java 17, Spring Web MVC, Data JPA, Security, AMQP, Flyway, jjwt |
| **Gateway prefix** | `/api/v1/therapist/` → service `/api/v1/` |

---

## 1. Purpose

The Therapist API is the **Booking bounded context** — the most business-logic-heavy service. It owns
the therapist directory, **patient↔therapist matching**, therapist **availability** (templates +
generated slots), **appointment booking** with **Zoom video** consultations, **clinical notes**, and
**reviews/ratings**. It also runs scheduled jobs and publishes booking events.

---

## 2. Domain model (booking DB)

| Table | Notes |
|---|---|
| `therapists` | directory: name, specialization, country, experience, `rating_avg`, `license_url`, matching attrs (`gender`, `is_lgbtq_allied`, `communication_style`, `treated_challenges[]`); `account_id` is an Auth-domain UUID |
| `therapist_zoom_credentials` | per-therapist **static Zoom Personal Meeting Room** (shared PK via `@MapsId`) |
| `weekly_templates` | recurring weekly availability patterns |
| `schedule_slots` | concrete bookable slots; `is_booked` flag |
| `appointments` | bookings; **snapshots** the Zoom meeting number/password at booking time; status machine |
| `clinical_notes` | one per appointment (`appt_id` unique): diagnosis, recommendations |
| `reviews` | one per appointment; recalculates `therapists.rating_avg` |
| `profiles_preferences` | patient matching intake (orientation, reasons[], communication style) |
| `therapist_assignments` | patient↔therapist pairing history: `ACTIVE` / `INACTIVE` / `CHANGED_BY_REQUEST` |

> **No cross-domain JPA joins.** `appointments.profile_id`, `therapists.account_id`,
> `therapist_assignments.profile_id` are scalar UUIDs referencing the Auth domain.

---

## 3. Key business rules (implemented)

### Booking (`POST /api/v1/bookings`)
- Requires an authenticated patient (`profileId` from JWT).
- **Atomically** locks the slot:
  `UPDATE schedule_slots SET is_booked=true WHERE slot_id=:id AND is_booked=false` — 1 row = success,
  0 rows = `409 Conflict` (someone else got it first).
- Creates an `Appointment` (default mode `VIDEO`, status `UPCOMING`), **snapshot-copies** the
  therapist's Zoom room onto the appointment for historical immutability.
- If `video.provider=zoom` and the therapist has no Zoom credential → `404`.
- Publishes `appointment.booked` to `booking.exchange`.

### Video join (`GET /api/v1/bookings/{id}/join`)
- Rejects if now < `start_datetime − 10 min` (`403`).
- Transitions `UPCOMING → IN_PROGRESS`.
- Returns `VideoJoinResponseDto(meetingNumber, password, sdkJwt)` — meeting details from the snapshot,
  `sdkJwt` minted at join time by the active video provider.

### Clinical notes (`POST /api/v1/notes`)
- Therapist/admin-only; only the **assigned** therapist; appointment must be `IN_PROGRESS`; one note
  per appointment. On success → appointment `IN_PROGRESS → COMPLETED`.

### Reviews (`POST /api/v1/reviews`)
- Patient-only; own appointment; appointment must be `COMPLETED`; one review per appointment.
  Recomputes `therapists.rating_avg` (avg of all reviews, 2 dp).

### Matching
- `POST /api/v1/matching/preferences` upserts `profiles_preferences`, computes matches, and
  **auto-assigns the top-ranked therapist**. Also publishes integration events:
  `profile.demographics.updated`, `tracking.mood.logged`, and `ai.crisis.alerted` (when self-harm
  risk is flagged).
- `findMatches` filters by communication style + optional strict LGBTQ-allied requirement, orders by
  overlap of requested reasons vs therapist `treated_challenges`; falls back to style-agnostic
  filtering if no exact style match.
- `assignTherapist` deactivates the current `ACTIVE` assignment and creates a new one.
- Every (de)activation publishes `therapist.assignment.changed` → `booking.exchange` after commit
  (`AssignmentEventPublisher`). Consumed by **auth-service** (assignment replica) and by
  **social-api**, which creates the therapist↔patient friendship + direct chat channel on `ACTIVE`
  (old chats are kept on re-match).

### Scheduled jobs
- **Generate slots:** `@Scheduled(cron "0 0 2 * * SUN", Asia/Ho_Chi_Minh)` — materialises active
  weekly templates into the next 30 days (idempotent; local→UTC conversion).
- **Cleanup slots:** `@Scheduled(cron "0 0 3 1 * *")` — deletes unbooked slots older than 30 days.

---

## 4. API surface (selected — full list in [04-API-Reference](../04-API-Reference/README.md))

| Area | Examples |
|---|---|
| Appointments (`AppointmentController`, `/api/v1`) | `POST /bookings`, `GET /bookings/{id}/join`, `POST /bookings/{id}/cancel|confirm|reject`, `GET /profiles/{profileId}/appointments/upcoming|history|unreviewed`, `GET /therapists/{therapistId}/appointments` |
| Availability (`AvailabilityController`, `/api/v1/therapists/{therapistId}`) | `GET/POST /slots`, `POST /slots:bulk`, `PUT/DELETE /slots/{slotId}`, CRUD `/availability-templates` |
| Therapists (`TherapistController`, `/api/v1/therapists`) | `GET /{id}`, `GET /{id}/slots`, `GET /{id}/reviews` |
| Matching (`TherapistMatchingController`, `/api/v1/matching`) | `POST /preferences`, `GET /therapists`, `POST /assign/{therapistId}` |
| Assignment (`TherapistAssignmentController`, `/api/v1/profiles`) | `GET /{profileId}/assigned-therapist` (self-or-admin) |
| Clinical notes (`ClinicalNoteController`, `/api/v1/notes`) | `GET /appointments/{appointmentId}`, `GET/PUT /{noteId}`, `POST /{noteId}/finalize` |
| Reviews (`ReviewController`, `/api/v1/reviews`) | `POST /` |
| Dashboard (`TherapistDashboardController`) | `GET /api/v1/therapists/{therapistId}/dashboard/summary` |
| Patients (`TherapistPatientController`) | `GET /api/v1/therapists/{therapistId}/patients`, patient tags/risk-level read/update |
| Internal | `GET /internal/assignments` |
| Test/utility | `POST /api/v1/test/trigger-generation|trigger-cleanup` (permit-all) |

---

## 5. Video provider architecture (pluggable)

- Abstracted behind `VideoConsultationProvider` (`VideoRoomDetailsDto getVideoRoomDetails(Therapist)`).
- Active provider via `video.provider` / `VIDEO_PROVIDER` (default `zoom`).
- **Zoom** (`ZoomVideoServiceImpl`): reads the therapist's static Personal Meeting Room from
  `therapist_zoom_credentials` and mints a Zoom SDK JWT. **Multi-account**: each therapist has a
  dedicated Zoom Basic account to dodge concurrent-meeting limits.
- **Jitsi** (`JitsiVideoServiceImpl`): returns a random room UUID, no password/JWT.
- Swapping providers = a new `@Service`; `BookingService` is unaffected (Dependency Inversion).
- The backend is only an **authorization gatekeeper** — media is peer-to-peer in the client Zoom SDK.

---

## 6. Security & error contracts

- Stateless JWT (RS256). Verification keys come from Auth's JWKS (`JWT_JWKS_URI`), resolved by `kid`
  through `JwksVerificationKeyLocator` on jjwt's `Locator` SPI — this repo has no `shared-jwt`
  dependency and parses with jjwt, so it could reuse neither the monorepo's provider nor Nimbus.
  Validates `iss`/`aud`; the optional `kid` pin is ignored under JWKS (it would break rotation).
- Principal = `profileId` claim (fallback `sub`); `TEEN → ROLE_PATIENT`, `THERAPIST → ROLE_THERAPIST`.
- `/api/v1/test/**` permit-all; everything else authenticated; method security for role checks.
- `GlobalExceptionHandler` → RFC `ProblemDetail`: `SlotAlreadyBooked → 409`, `ResourceNotFound → 404`,
  `MeetingNotOpen → 403`, `InvalidAppointmentState → 409`, `*AlreadyExists → 409`,
  `ReviewNotAllowed → 403`, validation → `400`, uncaught → `500`.

---

## 7. Run it

```bash
cd therapist-api && docker compose up -d --build
docker logs therapist-api-api-1 | grep "Started"   # no /actuator/health exposed
```

---

## 8. Known gaps & required-but-pending policies
From the in-repo `CONTEXT/CONTEXT.md`:
- Not yet implemented: booking lead-time enforcement, cancellation rule enforcement.
- **Required policies** (constraints to honour when extending): a slot must be booked ≥ **12h**
  before start; cancellation allowed only ≥ **24h** before start.
- Redis-backed refresh tokens (hybrid security) not yet implemented in this service.

In-repo deep references: `therapist-api/CONTEXT/CONTEXT.md`,
`therapist-web-ui/docs/THERAPIST_API_CONTROLLER_REFERENCE.md`, `therapist-web-ui/docs/Therapist_Features.md`.
