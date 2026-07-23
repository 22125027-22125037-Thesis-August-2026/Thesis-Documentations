# E2E-Mobile — End-to-End Test Plans for the uMatter Mobile App

> Phase 1. Real Android build, real device, deployed backend. Fifteen plans, ordered by the
> [Mobile App Showcase](../../08-Product-Showcase/Mobile-App-Showcase.md).

Before your first run: [00-Test-Strategy](../00-Test-Strategy-and-Conventions.md) ·
[01-Test-Environment](../01-Test-Environment-Builds-and-Data.md).

---

## The plans

| # | Plan | Pri | Cases | Est. |
|---|---|---|---|---|
| M01 | [Mood Check-In & Focus Mode](E2E-M01-Mood-Check-In-and-Focus-Mode.md) | P0 | 10 | 60 min + overnight |
| M02 | [Emotional Journal — Góc tâm tư](E2E-M02-Emotional-Journal.md) | P0 | 10 | 50 min |
| M03 | [Treasure Box — Hộp kho báu](E2E-M03-Treasure-Box.md) | P0 | 8 | 35 min |
| M04 | [Sleep & Nutrition Tracking](E2E-M04-Sleep-and-Nutrition-Tracking.md) | P1 | 9 | 45 min |
| M05 | [Automatic Step Counting](E2E-M05-Automatic-Step-Counting.md) | P1 | 8 | 45 min + walking |
| M06 | [Breathing & Meditation](E2E-M06-Breathing-and-Meditation.md) | P1 | 9 | 40 min |
| M07 | [AI Companion — Bạn Tâm Giao](E2E-M07-AI-Companion-Ban-Tam-Giao.md) | P0 | 10 | 50 min |
| M08 | [Therapist Matching & Booking](E2E-M08-Therapist-Matching-and-Booking.md) | P0 | 11 | 60 min |
| M09 | [Consultation Session & Aftercare](E2E-M09-Consultation-Session-and-Aftercare.md) | P0 | 10 | 60 min (2 clients) |
| M10 | [Social Friends & Real-Time Chat](E2E-M10-Social-Friends-and-Realtime-Chat.md) | P1 | 10 | 50 min (2 devices) |
| M11 | [Daily Trophy & Streaks](E2E-M11-Daily-Trophy-and-Streaks.md) | P1 | 8 | 40 min + multi-day |
| M12 | [Crisis Safety & Emergency Support](E2E-M12-Crisis-Safety-and-Emergency-Support.md) | P0 | 9 | 40 min |
| M13 | [Privacy — Data-Access Grants](E2E-M13-Privacy-Data-Access-Grants.md) | P0 | 10 | 60 min (2 clients) |
| M14 | [Notifications End-to-End](E2E-M14-Notifications-End-to-End.md) | P1 | 9 | 45 min |
| M15 | [First Run, Personalisation & Session Lifecycle](E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md) | P2 | 10 | 45 min |

---

## Recommended run order

The plans are **not** independent — some produce the data others need. Running them in this order
means you never have to seed by hand.

```
Day 0  (setup)        Register TEEN-A / TEEN-B / THERAPIST-T · verify therapist · generate slots
   │
   ├─ M15-01..03      Clean install → first-run tour → register/login   (do this first: needs virgin state)
   ├─ M01             Mood check-in           ─┐
   ├─ M02             Journal                  │ produce the tracking history
   ├─ M04             Sleep & nutrition        │ that M07 / M11 / M13 read
   ├─ M05             Steps (start early —    ─┘ needs real walking + a day boundary)
   ├─ M03             Treasure Box
   ├─ M06             Breathing & meditation
   ├─ M11-01..03      Trophy (needs ≥4 categories logged today — so run after M01/M02/M04/M05)
   ├─ M07             AI companion (grounding needs the history above)
   ├─ M12             Crisis safety + SOS
   ├─ M08             Matching & booking      ─┐
   ├─ M14             Notifications            │ M14 rides on M08's booking event
   ├─ M09             Session & aftercare     ─┘ needs M08's appointment
   ├─ M10             Social chat (2 devices)
   ├─ M13             Grants (needs M09's therapist + M10's friend + the history)
   └─ M11-04, M15-*   Streak & personalisation leftovers
Day +1 (next morning) M01-06..09 focus-mode delivery · M05-04 day rollover · M11-04 streak increment
```

**Two cases must be started on day 0 and checked on day 1**: `M01-07` (a Focus Mode nudge actually
arriving) and `M05-04` (the step baseline resetting across midnight). Neither can be faked by moving
the device clock without invalidating other results — plan for the overnight.

---

## Packs

Not every run needs all 15 plans. Three named subsets:

### 🚀 Smoke pack — ~25 min, run after every deploy
The minimum that proves the system is alive end-to-end.

`M15-03` (login) → `M01-01` (log a mood) → `M02-01` (journal entry) → `M07-01` (AI replies) →
`M08-06` (book a slot) → `M14-02` (booking notification arrives) → `M10-05` (chat message delivers).

If any of these fail, stop and fix — deeper testing on a broken build produces noise.

### 🎬 Demo-readiness pack — ~90 min, run the day before the defence
Everything the council will actually be shown, in demo order.

All **P0** cases from M01, M02, M03, M07, M08, M09, M12, M13 — plus `M14-02` and `M11-01`.
**Plus the grant-replica check**: create a fresh grant through the app and confirm the therapist can
read, because seeded grants do not work (see [01 §4](../01-Test-Environment-Builds-and-Data.md#4-test-accounts)).

### 🔁 Full regression pack — ~10 hours over two days
Every case in every plan. Run once per release candidate.

---

## Conventions used inside these plans

- **Vietnamese UI strings are quoted as the user sees them** (`"Cảm xúc lúc này"`), with an English
  gloss on first use. The app is bilingual; unless a case is about language, run in **Tiếng Việt**,
  because that is the demo language and where text-overflow problems live.
- **"Backend verification"** means an HTTP call or a cold app relaunch — never just "the screen shows
  it". See [00 §6](../00-Test-Strategy-and-Conventions.md#why-backend-verification-is-mandatory).
- **`{profileId}`** in a curl is the tester's own, from `GET /api/v1/auth/me`.
- A case marked **⚠️ known issue** documents behaviour that is already understood; it is verified to
  still behave as documented, and is *not* re-raised as a new defect.

---

## Run log (all plans)

| Date | Build | Pack | Result | Notes |
|---|---|---|---|---|
| — | — | — | — | — |
