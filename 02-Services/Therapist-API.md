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
generated slots), **appointment booking** with **Jitsi video** consultations, **clinical notes**, and
**reviews/ratings**. It also runs scheduled jobs and publishes booking events.

---

## 2. Domain model (booking DB)

| Table | Notes |
|---|---|
| `therapists` | directory: name, specialization, country, experience, `rating_avg`, `license_url`, matching attrs (`gender`, `is_lgbtq_allied`, `communication_style`, `treated_challenges[]`); `account_id` is an Auth-domain UUID |
| `therapist_zoom_credentials` | per-therapist static Zoom Personal Meeting Room (shared PK via `@MapsId`). **Unused under Jitsi** — kept for the dormant Zoom provider |
| `weekly_templates` | recurring weekly availability patterns |
| `schedule_slots` | concrete bookable slots; `is_booked` flag |
| `appointments` | bookings; **snapshots** the video room (a Jitsi room UUID) at booking time; status machine |
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
- Creates an `Appointment` (default mode `VIDEO`, status `UPCOMING`), **snapshot-copies** the video
  room onto the appointment for historical immutability — under Jitsi that is a freshly generated room
  UUID, unique per booking.
- Only on the dormant Zoom path: no Zoom credential for the therapist → `404`. Not reachable under
  `VIDEO_PROVIDER=jitsi`.
- Publishes `appointment.booked` to `booking.exchange`.

### Video join (`GET /api/v1/bookings/{id}/join`)
- Rejects if now < `start_datetime − 10 min` (`403`).
- Transitions `UPCOMING → IN_PROGRESS`.
- Returns `VideoJoinResponseDto(meetingNumber, password, sdkJwt)` — `meetingNumber` is the room
  snapshotted at booking time. **Under Jitsi `password` and `sdkJwt` are both `null`**; the fields
  exist for the dormant Zoom path, where `sdkJwt` would be minted at join time.

### Appointment status machine
`AppointmentStatus` = `REQUESTED`, `UPCOMING`, `IN_PROGRESS`, `PATIENT_COMPLETE`,
`PROFESSIONAL_COMPLETE`, `OVERALL_COMPLETE`, `CANCELLED`.

Since **V11** (July 2026) the old single `COMPLETED` is split three ways, because a patient review and
a therapist clinical note used to *each* independently flip the appointment straight to `COMPLETED` —
so "completed" could not tell you whether the note existed:

| Status | Meaning |
|---|---|
| `PATIENT_COMPLETE` | Patient submitted a review; therapist has **not** finalized a note yet |
| `PROFESSIONAL_COMPLETE` | Therapist finalized a note; patient has **not** reviewed yet |
| `OVERALL_COMPLETE` | Both sides are done |

`AppointmentStatus.isCompletionVariant()` is the canonical "is this finished?" test — never compare
against one literal. Both clients mirror it (`isCompletedAppointmentStatus()` /
`COMPLETED_APPOINTMENT_STATUSES` in `therapistApi.ts` and `lib/api/therapist.ts`).
`V11__split_completed_appointment_status.sql` backfills existing rows by looking up whether a review
and/or a `FINALIZED` clinical note exists (`appointments.status` is a plain `VARCHAR(50)` with no
CHECK constraint, so no DDL change was needed).

Everything that used to key off `COMPLETED` now spans all three: `…/appointments/history` returns the
three variants + `CANCELLED`, the dashboard's `completedThisMonth` counts all three, and cancellation
is refused once **any** completion variant is reached.

> ⚠️ **`NO_SHOW` has never been a value in this enum.** Spring fails the *whole* multi-value
> `@RequestParam` conversion if any one token doesn't match, so a client sending
> `status=OVERALL_COMPLETE,NO_SHOW` gets a `500` and loses **every** result, not just the no-shows.
> The therapist web UI's "Past" tab hit exactly this and silently showed no completed appointments.

### Clinical notes (`POST /api/v1/notes`)
- Therapist/admin-only; only the **assigned** therapist; one note per appointment.
- A `FINALIZED` note requires the appointment to be `IN_PROGRESS`, **or** `PATIENT_COMPLETE` within a
  **24-hour grace window** after the patient's review (`ClinicalNoteService.NOTE_GRACE_WINDOW_AFTER_PATIENT_COMPLETE`).
  Past that window → `ClinicalNoteNotAllowedException`; any other status →
  `InvalidAppointmentStateException`.
- On finalize: `IN_PROGRESS → PROFESSIONAL_COMPLETE`, or `PATIENT_COMPLETE → OVERALL_COMPLETE`.
- `DRAFT` notes bypass the state guard and do **not** move the appointment; they are rejected only
  once the therapist is locked out (`PROFESSIONAL_COMPLETE`, `OVERALL_COMPLETE`, `CANCELLED`).

### Reviews (`POST /api/v1/reviews`)
- Patient-only; own appointment; one review per appointment; at least **1 minute** after
  `start_datetime`; rejected for `UPCOMING` or `CANCELLED`.
- On success: `IN_PROGRESS → PATIENT_COMPLETE`, or `PROFESSIONAL_COMPLETE → OVERALL_COMPLETE`.
  Recomputes `therapists.rating_avg` (avg of all reviews, 2 dp).
- `GET …/appointments/unreviewed` lists **`PROFESSIONAL_COMPLETE`** appointments — that status *is*
  "note finalized, not yet reviewed"; the other two variants already carry a review.

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

- Abstracted behind `VideoConsultationProvider` (`VideoRoomDetailsDto getVideoRoomDetails(Therapist)`),
  selected by `video.provider` / `VIDEO_PROVIDER` via `@ConditionalOnProperty`.
- ✅ **Jitsi is the provider in use** — every deployed `.env` sets **`VIDEO_PROVIDER=jitsi`**.
  `JitsiVideoServiceImpl` returns a **random room UUID** with a `null` password and a `null` SDK JWT.
  The room is snapshotted onto the appointment at booking time, and **both clients** open
  `https://meet.jit.si/<room>` — the mobile app in a WebView (`VideoConsultationScreen`), the therapist
  web UI in `VideoSessionPage`. Nothing is self-hosted; `meet.jit.si` is a third-party service.
- ⚪ **Zoom (`ZoomVideoServiceImpl`) is implemented but not used.** It is the *code* default
  (`matchIfMissing = true`), so it runs only if `VIDEO_PROVIDER` is unset — which no deployment does.
  It would read the therapist's static Personal Meeting Room from `therapist_zoom_credentials` and mint
  a Zoom SDK JWT, one Zoom Basic account per therapist to dodge concurrent-meeting limits. Treat this
  as the demonstration that the abstraction holds, **not** as a description of the running system.
- Swapping providers = a new `@Service`; `BookingService` is unaffected (Dependency Inversion).
- The backend is only an **authorization gatekeeper** — it decides *whether* you may join and hands
  back a room; media never touches it.

> ⚠️ **`therapist_zoom_credentials` is dead weight under Jitsi.** The table and its `404`-if-missing
> rule only matter on the Zoom path. Under Jitsi the room is generated per booking and no therapist
> credential is consulted.

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
