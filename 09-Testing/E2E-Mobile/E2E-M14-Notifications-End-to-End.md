# E2E-M14 — Notifications End-to-End: Push, Email & Inbox

> **Showcase §6** — *"the moment you **book**, you get both a **push notification and an email**
> confirming the session, so it never slips past you."*
> **Plan priority: P1.** This plan traces one event across five systems — mobile → Therapist API →
> RabbitMQ → Notification API → FCM/SMTP → back to the phone. It is the widest fan-out in the
> architecture and the best single demonstration that the event-driven design actually works.

| | |
|---|---|
| **Scope** | FCM device-token registration/deregistration, the `appointment.booked` fan-out (inbox + email + push), foreground vs. background push, the in-app notification inbox, read/unread state, deep links |
| **Out of scope** | Local Focus Mode reminders — those are **on-device Notifee**, not push (→ [M01](E2E-M01-Mood-Check-In-and-Focus-Mode.md)); booking itself (→ [M08](E2E-M08-Therapist-Matching-and-Booking.md)) |
| **Services** | Auth, Therapist API (producer), RabbitMQ, Notification API (consumer), FCM, SMTP |
| **Screens** | `NotificationScreen`, `NotificationDetailScreen`, home bell badge |
| **Est. duration** | 45 min |
| **Cases** | 9 |

---

## ⚠️ Only one notification producer exists

**`appointment.booked` is the only live end-to-end notification flow in the system.** Everything else
that once looked like a notification is gone or was never wired:

| Flow | Status |
|---|---|
| `appointment.booked` (booking) | ✅ **Live** — inbox row **+ email + FCM push**, all three |
| Therapist `confirm` / `reject` / `cancel` | ❌ Publishes **no event** → no notification of any kind |
| `streak.milestone` (tracking) | ❌ Consumer **deleted** July 2026; nothing publishes it |
| `message.missed` (social) | ❌ Consumer **deleted** July 2026; `message_sent` publish is commented out |
| Focus Mode reminders | ⚠️ Not push at all — **local** Notifee scheduling on the device |

**Do not write, run, or file defects for the ❌ rows.** They are documented design state, not bugs
awaiting discovery. Testing them wastes a run and produces false failures.

---

## Preconditions

1. **Release build with `google-services.json` present** in `android/app/`. Without it, `getFcmToken()`
   returns `null`, login still succeeds, and every push case fails for an uninteresting reason. Check
   this first.
2. Notification permission granted on the device (Android 13+ prompts at runtime).
3. TEEN-A registered with a **mailbox you can read** (`+qa` addressing) — the email half of the fan-out
   is verified by actually opening the mail.
4. THERAPIST-T with bookable slots ([M08 preconditions](E2E-M08-Therapist-Matching-and-Booking.md)).
5. Access to the RabbitMQ management UI and notification-api logs, for `M14-05`.

---

## Test cases

### `E2E-M14-01` — Device token registers on login and deregisters on logout
**Priority** P1 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Fresh install, log in as TEEN-A | The notification permission prompt appears (Android 13+); grant it |
| 2 | Watch logcat during login | `[FCM] Registering device token (platform=ANDROID, profileId=…)` |
| 3 | Confirm registration server-side | The token is stored against TEEN-A's profile (`POST /api/v1/notification/devices`) |
| 4 | Log out | The app calls `DELETE /api/v1/notification/devices/{token}` |
| 5 | Log back in | The token is re-registered — re-posting the same token is idempotent and just refreshes it |
| 6 | Log in as TEEN-B **on the same device** | The token now belongs to B — A must not keep receiving B's pushes on this handset |

**Pass criteria** — The token registers on login, deregisters on logout, and follows the currently
signed-in account.

**Note** — Step 6 is a privacy check as much as a functional one. A stale token mapping means one
user's appointment notification lands on another user's phone. **S1** if it happens.

---

### `E2E-M14-02` — Booking produces a push notification 🔑
**Priority** P1 · **Type** Happy path · **Est.** 8 min

| # | Action | Expected result |
|---|---|---|
| 1 | With the app **backgrounded** (home button, not force-stopped), book a session ([M08-06](E2E-M08-Therapist-Matching-and-Booking.md)) — do the booking, then background immediately | — |
| 2 | Wait up to ~60 s | A push notification arrives in the shade |
| 3 | Read it | It concerns the booked appointment, in Vietnamese, with a meaningful title and body — no raw JSON, no `null`, no untranslated key |
| 4 | Check the timing | It fires at **booking** time — not on therapist confirmation |
| 5 | Tap it | The app opens; record where it lands (inbox, appointment detail, or home) |
| 6 | Repeat with the app **force-stopped** before booking completes | Push still arrives (FCM delivers to a stopped app on most devices — record OEM behaviour) |

**Pass criteria** — Booking produces a readable push within a minute, and tapping it opens the app.

**Failure triage** — In order: (1) is `google-services.json` in the build? (2) did the token register
(`M14-01`)? (3) did `appointment.booked` reach RabbitMQ (`M14-05`)? (4) is battery optimisation
throttling the app? Only after all four is it a defect in the notification pipeline.

---

### `E2E-M14-03` — Booking produces a confirmation email
**Priority** P1 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | After the booking in M14-02, open TEEN-A's mailbox | A confirmation email has arrived |
| 2 | Check the timing | Within a couple of minutes of booking |
| 3 | Read it | Correct therapist name, date, time (**local ICT**) and mode; Vietnamese renders correctly with diacritics |
| 4 | Check the sender and subject | Recognisable as uMatter, not a bare no-reply with an empty subject |
| 5 | Check the spam folder if nothing arrived | Record if it landed in spam — that is a real deliverability finding, not a pass |
| 6 | Check for template placeholders | No `{{name}}`, `null` or `undefined` anywhere in the body |

**Pass criteria** — A correct, well-formed confirmation email arrives in the inbox (not spam) with
accurate local-time details.

---

### `E2E-M14-04` — Booking produces an in-app inbox row
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open the notification inbox via the bell on Home | The booking notification is listed, newest first |
| 2 | Check the unread indicator | The bell badge and the row both reflect unread state |
| 3 | Verify server-side | `GET /api/v1/notification/notifications` contains the row; `…/unread-count` matches the badge |
| 4 | Open the notification detail | Full content renders correctly |
| 5 | Go back | The row is now marked **read**; the badge decrements |
| 6 | Cold restart the app | Read state persists — it was written server-side, not just locally |

**Pass criteria** — All three channels (push, email, inbox) fire for one booking, and inbox state is
server-backed.

**Note** — Cases M14-02, -03 and -04 are the same event observed three ways. Run them as one sequence
from a **single** booking; three separate bookings make correlating a failure much harder.

---

### `E2E-M14-05` — Trace the event through the pipeline
**Priority** P2 · **Type** Edge · **Est.** 6 min

The diagnostic case — run it once so that a future failure can be localised in minutes.

| # | Action | Expected result |
|---|---|---|
| 1 | Book a session and note the exact time (with timezone) | Recorded |
| 2 | Check therapist-api logs for the publish | `appointment.booked` published right after the appointment row is saved |
| 3 | Check RabbitMQ | The message passed through the exchange to the notification queue; queue depth returns to 0 |
| 4 | Check notification-api logs | The consumer handled it and fanned out to inbox + email + push |
| 5 | Confirm all three arrived | As in M14-02/03/04 |
| 6 | Record where each step's evidence lives | Future failures can then be bisected without re-deriving this |

**Pass criteria** — The event is traceable end to end, and the trace is written down.

---

### `E2E-M14-06` — Foreground notification handling
**Priority** P2 · **Type** Edge · **Est.** 5 min

When a push arrives while the app is **open**, FCM does not display it — the app must render it itself
(via Notifee on the `fcm-default` channel).

| # | Action | Expected result |
|---|---|---|
| 1 | Keep the app **open and in the foreground**, then trigger a booking notification (book from a second client for the same account, or use the Firebase console) | — |
| 2 | Observe | A notification is displayed — the app does not swallow it silently |
| 3 | Check the channel | It appears on the app's notification channel with a sensible icon |
| 4 | Tap it while the app is open | Handled without a crash or a duplicate screen |
| 5 | Confirm no duplicate | Exactly one notification is shown, not one from FCM plus one from Notifee |

**Pass criteria** — Foreground pushes are displayed exactly once and are tappable.

---

### `E2E-M14-07` — Inbox management: read all, mark unread, delete
**Priority** P2 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | With several unread notifications, use **mark all as read** | All rows become read; badge → 0 |
| 2 | Verify server-side | `…/unread-count` returns 0 |
| 3 | Mark one as **unread** again | Badge increments; the row shows unread |
| 4 | Delete a notification | It disappears; still gone after a cold restart |
| 5 | Pull to refresh on the inbox | Content refreshes without duplicating rows |
| 6 | Open the inbox with **no** notifications (fresh account) | A clean empty state |

**Pass criteria** — Every inbox action persists server-side and the badge stays consistent with the
unread count.

**Note** — The app keeps a local notification cache. Verify each action after a **cold restart**, not
just in-session, or you are testing the cache rather than the service.

---

### `E2E-M14-08` — Permission denied and offline behaviour
**Priority** P2 · **Type** Negative · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | **Deny** the notification permission at login | Login still succeeds — token retrieval is skipped gracefully (`[FCM] Notification permission not granted — skipping token fetch.`) |
| 2 | Use the app normally | No crashes; the inbox still works, since it is an API list not a push |
| 3 | Book a session with permission denied | **No** push (correct), but the **inbox row and email still arrive** — the channels are independent |
| 4 | Grant the permission in Android settings and restart the app | The token registers and pushes begin working |
| 5 | Open the inbox offline | A clear error/empty state, not an infinite spinner |

**Pass criteria** — Denied permission degrades to "no push" only; the other two channels are
unaffected and the app never crashes.

---

### `E2E-M14-09` — Confirm the absent flows are absent
**Priority** P2 · **Type** Negative · **Est.** 5 min

This case makes the documented gaps **verified on this build**, so nobody claims them on stage.

| # | Action | Expected result / record |
|---|---|---|
| 1 | Have THERAPIST-T **confirm** the appointment on the web UI | **No** push, **no** email, **no** inbox row for the patient — confirm and record |
| 2 | Have THERAPIST-T **cancel** it | Same — nothing |
| 3 | Reach a journal streak milestone | **No** notification — the `streak.milestone` consumer was deleted |
| 4 | Message an **offline** friend from TEEN-B ([M10](E2E-M10-Social-Friends-and-Realtime-Chat.md)) | **No** push to the offline recipient |
| 5 | Record all four | — |

**Pass criteria** — All four produce nothing, exactly as documented. **A notification appearing here
would be the surprise**, and would mean the documentation is out of date.

**Why this case exists** — Showcase §6's implementation note already concedes that confirm/reject/
cancel publish no event, and §11's "push" claims are broader than the code. Verifying the absence,
with a date and a build, is what lets the thesis state the scope of FR8 honestly instead of
overclaiming.

---

## Exit criteria for M14

- `M14-01` passes — tokens follow the signed-in account.
- **`M14-02`, `M14-03`, `M14-04` all pass from a single booking** — the three-channel fan-out is the
  showcased behaviour.
- `M14-09` executed and recorded, so the documented gaps are confirmed on this build.
- No **S1** defect open against token/account mapping.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| `appointment.booked` is the only live producer | ✅ Known and documented |
| It fires at **booking**, not therapist confirmation | ✅ Known |
| Therapist confirm/reject/cancel produce nothing | ⚠️ Known gap — confirm in `M14-09`, do not file |
| `streak.milestone` / `message.missed` consumers deleted | ⚠️ Known — those flows do not exist |
| Focus Mode reminders are local Notifee, not FCM | ✅ By design — they work offline and cost no push quota |
| A missing `google-services.json` silently disables push | ⚠️ Build-configuration trap — check before testing |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
