# E2E-M09 — Consultation Session & Aftercare

> **Showcase §6.3–6.4** — *"Meet by video or chat — 'Sẵn sàng kết nối với chuyên gia.' Join a secure
> video room, or message in text, right from the app."* then *"After the session, read your
> therapist's notes (diagnosis & recommendations), see your next steps, and leave a rating & review."*
> **Plan priority: P0.** The highest-stakes workflow in the product, and the only one that requires a
> **second human on a second client** to test. Budget accordingly.

| | |
|---|---|
| **Scope** | The 10-minute join window, video session (Jitsi in a WebView), text/chat consultation, session end, clinical-note retrieval, rating & review submission, therapist rating recomputation |
| **Out of scope** | Creating the appointment (→ [M08](E2E-M08-Therapist-Matching-and-Booking.md)), notifications (→ [M14](E2E-M14-Notifications-End-to-End.md)), the therapist's view of tracking data (→ [M13](E2E-M13-Privacy-Data-Access-Grants.md)) |
| **Services** | Therapist API, Social (for text consultation channels), Jitsi (`meet.jit.si`, external) |
| **Screens** | `WaitingRoomScreen`, `VideoConsultationScreen`, `ConsultationFeedbackScreen`, `AppointmentsHistoryScreen`, plus the **therapist web UI** |
| **Est. duration** | 60 min, **two operators** (phone + browser) |
| **Cases** | 10 |

---

## Preconditions

1. An appointment created in [M08-06](E2E-M08-Therapist-Matching-and-Booking.md), with its
   `appointmentId` recorded.
2. **A slot starting within the next ~15 minutes**, so the join window opens during the test. If the
   only available slots are far out, create a schedule template covering the current hour and re-run
   `POST /api/v1/test/trigger-generation`.
3. THERAPIST-T signed in to the **therapist web UI** in a desktop browser, on the same appointment.
4. **Camera and microphone permissions** ready to grant on the phone.
5. Good network on both ends — video is the most bandwidth-sensitive thing the app does.
6. Headphones on at least one end, or physical separation, to avoid audio feedback.

## Architecture notes that shape these tests

- **Video runs inside a WebView pointing at `https://meet.jit.si`** — a third-party service, not
  self-hosted. The app injects JavaScript to skip the pre-join page and to hide/dismiss Jitsi's
  auth prompts. Those injections target **CSS classes and `data-testid`s that Jitsi changes between
  releases**, so a login dialog appearing mid-demo is a realistic external failure mode. `M09-04`
  targets it specifically.
- **Join is gated twice**: the client checks status ∈ {`UPCOMING`, `IN_PROGRESS`} and the 10-minute
  window, and the server is asked `GET /api/v1/therapist/bookings/{id}/join` before entering.
- **Text consultation** rides the Social chat channel of type `THERAPIST_CONSULT`, so it shares the
  STOMP path tested in [M10](E2E-M10-Social-Friends-and-Realtime-Chat.md).

---

## Test cases

### `E2E-M09-01` — The join window opens
**Priority** P0 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open the appointment ~15 minutes before its start | Waiting room shows the therapist, date/time and mode; join is **disabled** with the helper text about the 10-minute rule |
| 2 | Wait until exactly 10 minutes before the start | The join control **becomes enabled** — without needing to leave and re-enter the screen, if the screen refreshes; otherwise re-enter and confirm |
| 3 | Note the exact time the button enabled | Recorded — it should be 10 minutes before, not 10 minutes after or 7 hours off |
| 4 | Confirm the therapist side | The web UI shows the same appointment as joinable |

**Pass criteria** — The join window opens exactly 10 minutes before the appointment's local start
time.

**Note** — Step 3 is a timezone check in disguise. If the button enables at the wrong time, compare
the slot's `startDatetime` (UTC on the wire) against the displayed local ICT time before blaming the
window logic.

---

### `E2E-M09-02` — Join a video consultation
**Priority** P0 · **Type** Happy path · **Est.** 10 min

| # | Action | Expected result |
|---|---|---|
| 1 | Inside the window, tap join | The server join check runs (`…/bookings/{id}/join`), then the video screen opens |
| 2 | Grant camera and microphone permissions when prompted | Permissions granted; the local video preview appears |
| 3 | Observe the entry | The Jitsi **pre-join page is skipped** — the user lands directly in the room |
| 4 | Have the therapist join from the web UI | Both participants see and hear each other |
| 5 | Check two-way audio and video explicitly | Speak from each end and confirm the other hears; wave at each camera |
| 6 | Use the in-call controls | Mute/unmute, camera on/off, and speaker toggle all work |
| 7 | Check the appointment status | It transitions to `IN_PROGRESS` |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/therapist/bookings/{appointmentId}" \
  -H "Authorization: Bearer $TOKEN" | jq '.status'
```

**Pass criteria** — Both parties are in the same room with working two-way audio and video, and the
appointment reaches `IN_PROGRESS`.

---

### `E2E-M09-03` — Denying camera/microphone permission
**Priority** P1 · **Type** Negative · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Join a session and **deny** camera permission | The user is told what is missing; the session does not silently continue with a black screen and no explanation |
| 2 | Deny the microphone too | Same — an explained state |
| 3 | Grant permissions from Android settings and rejoin | The session works without reinstalling |
| 4 | Join with camera denied but microphone allowed | Audio-only participation works, or the limitation is explained |

**Pass criteria** — Missing permissions produce an explanation and are recoverable in-session or on
rejoin.

---

### `E2E-M09-04` — Jitsi third-party surface behaves ⚠️
**Priority** P1 · **Type** Edge · **Est.** 6 min

The app injects CSS/JS into `meet.jit.si` to hide the pre-join screen and dismiss auth prompts. That
injection is coupled to Jitsi's markup, which changes without notice.

| # | Action | Expected result |
|---|---|---|
| 1 | Join a session and watch the **first 5 seconds** closely | No "I am the host" / login / "waiting for authentication" dialog is visible, even briefly |
| 2 | Check for a moderator-required prompt | The patient can be in the room without a moderator, or the wait is explained |
| 3 | Check the UI language | Record whether Jitsi renders in Vietnamese or English — the app tries to influence this, and English chrome inside a Vietnamese-first app is worth noting |
| 4 | Rotate the device during a call | Layout adapts; the call survives |
| 5 | Re-run this case **the day before any demo** | Still clean — external markup may have changed since the last run |

**Pass criteria** — No third-party auth dialog reaches the user, and the room renders correctly.

**Note** — This case cannot be "fixed once". It is an external dependency check with a short shelf
life. If a dialog does appear, that is **S2** for a demo and the mitigation is to re-verify the
injected selectors against the current Jitsi release.

---

### `E2E-M09-05` — Text/chat consultation
**Priority** P0 · **Type** Happy path · **Est.** 8 min

**Preconditions** — An appointment booked with mode **Chat/Text** (book a second one in M08 if needed).

| # | Action | Expected result |
|---|---|---|
| 1 | Join the text consultation inside the window | A chat room opens, identified as a therapist consultation (`THERAPIST_CONSULT`) |
| 2 | Send a message from the phone | It appears immediately on the patient side |
| 3 | Confirm on the therapist web UI | The message arrives **in real time**, without a manual refresh |
| 4 | Reply from the web UI | The reply appears on the phone in real time |
| 5 | Exchange several messages both ways | Ordering is correct on both sides; no duplicates; no missing messages |
| 6 | Background the phone app, have the therapist send a message, foreground again | The message is there (delivered live or fetched on resume) |
| 7 | Leave and re-enter the room | Full history loads in the right order |

**Backend verification** — `GET /api/v1/social/chats/channels/{channelId}/messages` matches what both
clients displayed.

**Pass criteria** — Real-time two-way messaging works between phone and web, and history is complete
after a reload.

**Failure triage** — If nothing arrives in real time, check the app's own log line
`STOMP connected successfully` before anything else. A WebSocket **upgrade** can succeed while STOMP
fails one layer above it — that exact failure once made chat silently non-functional in production for
months. See [01 §8](../01-Test-Environment-Builds-and-Data.md#8-diagnosing-not-guessing).

---

### `E2E-M09-06` — Ending the session
**Priority** P0 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | End the call from the phone | The app leaves the room and navigates onward (toward feedback), not to a blank screen |
| 2 | Confirm media is released | Camera indicator off, microphone off, no audio still playing |
| 3 | Check the appointment status | Moves toward `COMPLETED` (recording which side drives the transition — patient leaving, therapist ending, or a server rule) |
| 4 | Repeat, ending from the **therapist** side instead | The phone is informed — it does not sit in an empty room indefinitely |
| 5 | Repeat, killing the app mid-call | On relaunch the state is sane; the appointment is not stuck in `IN_PROGRESS` forever |

**Pass criteria** — Ending from either side releases media and moves the appointment out of
`IN_PROGRESS`.

**Note** — Step 5's outcome matters for the demo: a session stuck in `IN_PROGRESS` can block a rerun
of the whole workflow on the same account. Record the recovery path.

---

### `E2E-M09-07` — Read the therapist's clinical note
**Priority** P1 · **Type** Cross-client · **Est.** 8 min

| # | Action | Expected result |
|---|---|---|
| 1 | On the **web UI**, have THERAPIST-T write a clinical note for the completed appointment (diagnosis + recommendations) and save | Saved confirmation on the web side |
| 2 | On the phone, open the consultation's feedback/detail screen | The note loads (`GET …/clinical-notes/appointment/{id}` equivalent) |
| 3 | Compare content | Diagnosis and recommendations match **exactly** what the therapist wrote — no truncation, no lost line breaks, no mangled Vietnamese diacritics |
| 4 | Check the note's timestamp | Displayed in local ICT and consistent with when it was written |
| 5 | Open the screen **before** any note exists | A clean "no note yet" state — not an error and not a blank panel |
| 6 | Have the therapist **edit** the note, then reload on the phone | The updated text appears |

**Pass criteria** — The clinical note round-trips from web to phone with its content intact, and the
absent-note state is handled.

**Note** — Diacritics are worth checking character by character on at least one note. A note reading
"lo âu" as "lo âu" is fine; "lo Ã¢u" is an encoding defect (**S3**) that will be noticed instantly by
a Vietnamese-speaking examiner.

---

### `E2E-M09-08` — Leave a rating and review
**Priority** P1 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | After a `COMPLETED` session, open the feedback screen | A star rating control and a comment field are shown, alongside the session summary |
| 2 | Try to submit with **no** rating | Blocked with an explanation — a review without a rating is meaningless |
| 3 | Select 5 stars and write `QA M09 — buổi tư vấn rất hữu ích.` | Both accepted |
| 4 | Submit | Success confirmation; the screen switches to a read-only/submitted state |
| 5 | Try to submit **again** for the same appointment | Prevented — duplicate reviews are rejected |
| 6 | Verify on the therapist's public profile | The review appears in `GET …/therapists/{id}/reviews` and in the app's review list |

**Backend verification** — `POST /api/v1/therapist/reviews` returned success; the review is present in
the therapist's review list with the correct rating and comment.

**Pass criteria** — A rating + comment is stored once per appointment and becomes visible on the
therapist's profile.

---

### `E2E-M09-09` — Therapist's aggregate rating updates
**Priority** P1 · **Type** Cross-client · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Note the therapist's rating **before** the review (e.g. 4.0 from 2 reviews) | Recorded |
| 2 | Submit the 5-star review from M09-08 | — |
| 3 | Reopen the therapist's detail on the phone | The aggregate rating and review count have **both** updated |
| 4 | Recompute by hand | The new average matches `(sum + 5) / (count + 1)` |
| 5 | Check the therapist web UI | The same figure is shown there |

**Pass criteria** — The aggregate rating is recomputed correctly and consistently on both clients.

---

### `E2E-M09-10` — Reviewing is only possible for completed sessions
**Priority** P2 · **Type** Negative · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Try to review an **upcoming** appointment | Not offered / rejected |
| 2 | Try to review a **cancelled** appointment | Not offered / rejected |
| 3 | Check the unreviewed-appointments list | It contains only `COMPLETED` appointments without a review |
| 4 | After reviewing, check the list again | The appointment has dropped off it |
| 5 | Open a previously reviewed appointment | It opens in a read-only state showing the submitted review |

**Pass criteria** — Only completed, unreviewed appointments can be reviewed, and the unreviewed list
stays accurate.

---

## Exit criteria for M09

- `M09-01`, `M09-02` and `M09-06` pass — join, converse, end.
- `M09-05` passes — text consultation is genuinely real-time in both directions.
- `M09-07` passes — the clinical note reaches the patient intact.
- `M09-04` re-run within 24 h of any demo.
- No **S1** defect open against joining or ending a session.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| Video is `meet.jit.si`, a public third-party service | ✅ By design in this build — note the privacy implication for the thesis, since session media leaves the project's own infrastructure |
| Jitsi UI injection targets classes/`data-testid`s that shift between releases | ⚠️ Known fragility — `M09-04` exists for it |
| Therapist confirm/reject actions send no notification | ⚠️ Known gap (see [M08](E2E-M08-Therapist-Matching-and-Booking.md)) |
| Join is limited to 10 minutes before the start | ✅ By design |
| Text consultations use the Social chat channel | ✅ By design — shares the STOMP path with [M10](E2E-M10-Social-Friends-and-Realtime-Chat.md) |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
