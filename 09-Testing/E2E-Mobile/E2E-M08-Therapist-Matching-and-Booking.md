# E2E-M08 — Therapist Matching & Booking

> **Showcase §6.1–6.2** — *"Smart matching (ghép nối). An 8-step, caring intake… The system then pairs
> you with a compatible, verified professional."* then *"Book a session in a tap."*
> **Plan priority: P0.** This is the bridge from self-help to professional care — the part of uMatter
> that involves a real clinician and a real appointment. It is also the only workflow that produces a
> live notification event, so [M14](E2E-M14-Notifications-End-to-End.md) depends on it.

| | |
|---|---|
| **Scope** | The 8-step matching intake, per-step validation, submission, therapist discovery & detail, available slots, booking a session, appointment list/detail, cancellation with a reason |
| **Out of scope** | Joining and running the session (→ [M09](E2E-M09-Consultation-Session-and-Aftercare.md)), the booking notification (→ [M14](E2E-M14-Notifications-End-to-End.md)), granting the therapist data access (→ [M13](E2E-M13-Privacy-Data-Access-Grants.md)) |
| **Services** | Auth, Therapist API (`/api/v1/therapist/*`), Notification (indirectly, via `appointment.booked`) |
| **Screens** | `TherapistBookingLanding`, `MatchingFormScreen`, `TherapistDetailScreen`, `BookingScreen`, `ConsultationDetailScreen`, `AppointmentsHistoryScreen`, `WaitingRoomScreen` |
| **Est. duration** | 60 min |
| **Cases** | 11 |

---

## Preconditions

1. TEEN-A logged in, **with no prior matching submission** for the first case (register fresh if you
   have already matched on this account — the intake is a first-run experience).
2. **THERAPIST-T fully prepared** ([01 §4](../01-Test-Environment-Builds-and-Data.md#therapist-setup-needed-before-m08)):
   registered, profile complete, **verified**, schedule template created, and slots materialised:
   ```bash
   curl -X POST https://umatter-apcs.duckdns.org/api/v1/test/trigger-generation
   ```
3. Confirm slots are visible to the mobile client before starting:
   `GET /api/v1/therapist/therapists/{therapistId}/slots?page=0&size=200&sort=startDatetime,asc`
4. At least one slot **in the future but today** for the join-window cases in M09.

## Appointment state machine

| Status | Meaning |
|---|---|
| `REQUESTED` | Booked by the patient, awaiting therapist action |
| `UPCOMING` | Confirmed / scheduled |
| `IN_PROGRESS` | Session under way |
| `COMPLETED` | Session finished |
| `CANCELLED` | Cancelled by either side |

⚠️ **The `appointment.booked` event fires at booking time**, from `BookingService`, right after the
row is saved — **not** when the therapist confirms. The therapist's `confirm`/`reject`/`cancel`
actions publish **no** event and therefore generate no notification. Do not write an expected result
that says otherwise.

---

## Test cases

### `E2E-M08-01` — Complete the 8-step matching intake
**Priority** P0 · **Type** Happy path · **Est.** 8 min

| # | Action | Expected result |
|---|---|---|
| 1 | From the Therapist tab, start matching (*"ghép nối"*) | Step **1/8** with a progress bar at 12.5 % |
| 2 | **Step 1** — *"Đã từng đi trị liệu bao giờ chưa?"* — choose *"Chưa bao giờ"* | Selectable; Next enables |
| 3 | **Step 2** — *"Giới tính của bạn là gì?"* + age — choose *"Nữ"*, enter age `17` | Both required; Next stays disabled until **both** are set |
| 4 | **Step 3** — sexual orientation — choose any (incl. *"Không muốn nói"*) | Opting out is a valid answer that lets you continue |
| 5 | **Step 4** — LGBTQ+-allied therapist preference — choose *"Có"* or *"Không"* | Next enables |
| 6 | **Step 5** — self-harm question — choose an answer | Next enables (see `M08-03` for the "Có" path) |
| 7 | **Step 6** — *"Điều gì khiến bạn tìm đến trị liệu hôm nay?"* — select **two** reasons | Multi-select works; Next requires ≥ 1 |
| 8 | **Step 7** — mood scales (*Lo lắng*, *Mất hứng thú*, *Mệt mỏi*) | Sliders/scales move between *Ít* and *Nhiều*; this step is **optional** — Next is enabled even untouched |
| 9 | **Step 8** — communication style — choose *"Kết hợp cả hai"* | Next becomes Submit |
| 10 | Submit | Loading, then you land on the **Therapist tab** with matching results |

**Backend verification**
```bash
# preferences were saved
curl -s -X POST https://umatter-apcs.duckdns.org/api/v1/therapist/matching/preferences ...
# and re-open the app: an assigned therapist should now resolve
curl -s "https://umatter-apcs.duckdns.org/api/v1/therapist/..." -H "Authorization: Bearer $TOKEN"
```
Use the app's own "active assigned therapist" view as the functional check.

**Pass criteria** — All 8 steps complete, submission succeeds, and the app returns to the therapist
tab with a matched/assigned professional.

---

### `E2E-M08-02` — Per-step validation and back navigation
**Priority** P1 · **Type** Negative · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | On each of steps 1–6 and 8, try to advance **without** answering | Next is disabled (or blocked) — no step can be skipped except step 7 |
| 2 | On step 2, enter an age but no gender | Still blocked — both fields are required |
| 3 | On step 2, enter a non-numeric or absurd age (`abc`, `0`, `999`) | Record the behaviour; the field should not accept nonsense silently |
| 4 | Advance to step 5, then press Back twice | Returns to step 3 with **previous answers still selected** |
| 5 | Change an earlier answer, then go forward again | Later answers are preserved; the progress bar tracks correctly (step/8) |
| 6 | On step 1, press Back | Leaves the form entirely (back to the previous screen) |
| 7 | Re-enter the form after leaving mid-way | Record whether progress resumes or restarts — either is acceptable if it does not lose a submitted answer |

**Pass criteria** — No mandatory step can be bypassed, and back-navigation preserves answers.

---

### `E2E-M08-03` — Self-harm answer "Có" — safety response 🚨
**Priority** P0 · **Type** Edge · **Est.** 5 min

> Step 5 asks: *"Hiện tại, bạn có đang suy nghĩ về việc làm hại bản thân hoặc người khác không?"*
> Showcase §11 states, on the advice of the physician consulted for the project, that *"an app for
> vulnerable teens must never leave someone alone in a crisis."* This case tests exactly that moment.

| # | Action | Expected result |
|---|---|---|
| 1 | Reach step 5 and read the subtitle | It reads *"Nếu bạn cần hỗ trợ khẩn cấp, hãy liên hệ dịch vụ y tế ngay lập tức."* — present, but **static text** |
| 2 | Answer **"Có"** (yes) | **Record precisely what happens.** In the current build the form simply enables Next and continues — no hotline is surfaced, no support screen is offered, no confirmation is shown |
| 3 | Continue and submit | The answer is stored in the matching preferences |
| 4 | Check whether anything downstream reacts | Record whether the therapist sees this flagged, and whether any alert/notification is raised |

**Pass criteria** — This case documents the actual behaviour. It **passes** only if the behaviour is
recorded and reviewed by the team.

**⚠️ Finding to escalate, not to bury.** A teen disclosing active self-harm ideation and receiving no
response beyond a "Next" button is a **safety gap**, and it is inconsistent with the app's own crisis
posture elsewhere: the AI companion surfaces a hotline on crisis keywords ([M12](E2E-M12-Crisis-Safety-and-Emergency-Support.md)),
and the SOS area is one tap away. Recommended remediation (a product decision, not a test verdict):
answering "Có" should surface the same crisis resources the AI does — hotline **1800 599 920** and the
emergency-support screen — before the form continues. Raise as **S2 minimum**; argue for S1 if the
council is likely to probe safety claims.

---

### `E2E-M08-04` — Matching produces a verified, plausible therapist
**Priority** P0 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | After submitting, view the assigned/matched therapist | A therapist is shown — not an empty state |
| 2 | Check the therapist is **verified** | Only verified professionals are offered |
| 3 | If LGBTQ+-allied preference was "Có", check the match | Record whether the preference influenced the result |
| 4 | Compare against the therapist list | `GET /api/v1/therapist/therapists` — the match is a real member of that list |
| 5 | Re-submit the intake with **different** answers (fresh account) | Record whether the match changes — evidence that preferences affect matching at all |

**Pass criteria** — Matching returns a real, verified therapist. Step 5's result is recorded, because
"smart matching" is a claim the council may test by asking *how* it matches.

---

### `E2E-M08-05` — Therapist discovery and detail
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open the therapist list | Therapists render with name, specialisation, rating and avatar; no broken images |
| 2 | Open a therapist's detail | Bio, specialties, languages, rating and reviews load |
| 3 | Open the reviews | `GET …/therapists/{id}/reviews` content matches what is displayed; an unreviewed therapist shows a clean empty state, not `0.0 ★` styled as a bad score |
| 4 | Go back and open a different therapist | Content updates fully — no data bleeding from the previous therapist |

**Pass criteria** — Detail and reviews load correctly and independently per therapist.

---

### `E2E-M08-06` — Book a session
**Priority** P0 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | From the therapist's detail, open booking | The calendar/slot view loads available slots |
| 2 | Check the slots against the API | Only genuinely **free**, future slots are offered — no past slots, no already-booked ones |
| 3 | Pick a date and a time | Selection is visually confirmed |
| 4 | Choose the mode (**Video** or **Chat/Text**) | Selectable |
| 5 | Enter a reason (optional) | Accepted |
| 6 | Confirm the booking | Success feedback; you are taken to the confirmation / waiting-room view |
| 7 | Note the returned `appointmentId` | Recorded for M09 and M14 |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/therapist/bookings/{appointmentId}" \
  -H "Authorization: Bearer $TOKEN" | jq '{status, mode, startDatetime, therapistId, reason}'
```
Status is `REQUESTED` or `UPCOMING`; mode and start time match what was chosen.

**Pass criteria** — The booking is created server-side with the exact slot and mode chosen, and the
app confirms it.

**Chain** — This appointment is the input to [M09](E2E-M09-Consultation-Session-and-Aftercare.md) and
[M14](E2E-M14-Notifications-End-to-End.md). Record the id.

---

### `E2E-M08-07` — The booked slot is consumed
**Priority** P1 · **Type** Edge · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Immediately after M08-06, reopen the same therapist's slot list | The booked slot is **no longer offered** |
| 2 | Query the slots API directly | The slot is absent or flagged as taken |
| 3 | From a **second** account (TEEN-B), open the same therapist's slots | The slot is not offered there either |
| 4 | Attempt to book the same slot twice (rapid double-tap on confirm) | Exactly **one** appointment is created — no duplicate booking |

**Pass criteria** — A booked slot disappears for everyone, and double-submission cannot create two
appointments.

**Note** — Step 4 is the classic demo-day failure. Tap the confirm button twice quickly; if two
appointments appear, that is **S2** and will happen on stage.

---

### `E2E-M08-08` — Booking failure paths
**Priority** P1 · **Type** Negative · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Enable aeroplane mode and confirm a booking | Clear failure message; no local "success" state |
| 2 | Restore network; check the backend | **No** phantom appointment was created |
| 3 | Book a slot that a second account books first (race) | The loser gets a clear "slot no longer available" message, not a generic 500 |
| 4 | Try to book with a therapist who has **no** slots | Empty-state message, not a crash or an endless spinner |

**Pass criteria** — Failures are explained and never leave a partial or phantom appointment.

---

### `E2E-M08-09` — Appointment list and detail
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open the appointments view | The new appointment is listed with a status badge |
| 2 | Check the status badge | Matches the API's status (`REQUESTED`/`UPCOMING`) and is human-readable in Vietnamese |
| 3 | Open the detail | Therapist name, specialisation, date/time, mode and reason all match the booking |
| 4 | Check the displayed time | Local ICT time matching the slot's `startDatetime` — **check the timezone conversion explicitly**, a UTC/ICT slip shows as a 7-hour error |
| 5 | Open the appointment history | Past `COMPLETED`/`CANCELLED` appointments appear separately from upcoming ones |
| 6 | Cold restart and re-check | Everything still correct — read from the server |

**Pass criteria** — The appointment appears with correct details and a **correct local time**.

---

### `E2E-M08-10` — Cancel an appointment with a reason
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open an upcoming appointment and choose cancel | A reason field appears (max 1 000 characters) |
| 2 | Try to cancel with an **empty** reason | Record whether a reason is required |
| 3 | Enter `QA test — kiểm thử huỷ lịch hẹn` and confirm | A confirmation step is required before the cancellation goes through |
| 4 | Confirm | Status becomes `CANCELLED`; the appointment moves to history |
| 5 | Verify the backend | `status: "CANCELLED"`, `cancellationReason` matches, `cancelledAt` set |
| 6 | Check the slot | Record whether the freed slot returns to the therapist's availability |
| 7 | Check for a notification | **None expected** — cancellation publishes no event (see the note above). Absence here is correct |

**Pass criteria** — Cancellation persists with its reason and moves the appointment to history.
**Do not** raise "no notification on cancel" as a defect.

---

### `E2E-M08-11` — Waiting-room state before the join window
**Priority** P1 · **Type** Edge · **Est.** 5 min

Joining is allowed only when the status is `UPCOMING` or `IN_PROGRESS` **and** the current time is
within **10 minutes** before the appointment.

| # | Action | Expected result |
|---|---|---|
| 1 | Open a booked appointment more than 10 minutes ahead | The waiting room shows the appointment; the join button is **disabled** |
| 2 | Read the helper text | *"Bạn chỉ có thể tham gia trong vòng 10 phút trước giờ hẹn."* |
| 3 | Tap the disabled join button | Nothing happens (or an explanation) — no navigation into an empty room |
| 4 | Check a `CANCELLED` appointment's waiting room | Join is unavailable and the state is clear |
| 5 | Wait until inside the 10-minute window (or book a near-term slot) | The join button becomes enabled — verified in [M09-01](E2E-M09-Consultation-Session-and-Aftercare.md) |

**Pass criteria** — The join window is enforced client-side with an explanation, and no status outside
`UPCOMING`/`IN_PROGRESS` can join.

---

## Exit criteria for M08

- `M08-01`, `M08-04`, `M08-06` pass — intake, matching and booking work end-to-end.
- `M08-03` is **executed and its finding escalated** — this is a safety item, not an optional case.
- `M08-07` passes — no double-booking.
- `M08-09` step 4 passes — times display in correct local time.
- No **S1** defect open against booking creation.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| `appointment.booked` fires at **booking** time, not on therapist confirmation | ✅ Known and documented |
| Therapist confirm / reject / cancel publish **no** event → no notification | ⚠️ Known gap — do not file |
| Answering "Có" to self-harm produces no in-app safety response | 🚨 **Verify and escalate** — see `M08-03` |
| Step 7 (mood scales) is optional | ✅ By design (`canProceed` returns true) |
| The slots endpoint is `/therapists/{id}/slots`, paged | ✅ Informational, for backend verification |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
