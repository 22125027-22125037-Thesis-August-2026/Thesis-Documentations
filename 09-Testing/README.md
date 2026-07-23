# 09 — Testing

> The uMatter test suite: strategy, environment, and the individual test plans.
> **Phase 1 (in progress): end-to-end testing of the teen mobile app, workflow by workflow.**

This folder is where all *deliberate, written-down* testing for uMatter lives. It is separate from
[06-Development/03-Testing-and-Accounts](../06-Development/03-Testing-and-Accounts.md), which is a
short developer-facing note on seeded accounts and smoke curls. **That page tells you how to poke the
system; this folder tells you how to prove it works.**

---

## 🎯 Priority: complete user workflows, not input boxes

The ordering principle for everything in this folder:

1. **End-to-end, on a real phone, first.** The thing being graded is a working product. A test that
   drives the actual Android build against the deployed backend is worth ten unit tests of a mapper.
2. **Complete workflows over isolated widgets.** We do **not** spend a test plan on "the email field
   rejects `abc`" or "the password toggle shows the password". Field-level validation is covered
   *incidentally*, inside the workflow that needs it, and only where a failure would actually block
   the user.
3. **Showcase features rank highest.** The features in
   [08-Product-Showcase/Mobile-App-Showcase](../08-Product-Showcase/Mobile-App-Showcase.md) are what
   the council will be shown and what the thesis claims. Test-plan numbering follows the showcase's
   own section order, so §1 (mood check-in) is `E2E-M01`, §3 (Treasure Box) is `E2E-M03`, and so on.
4. **The "connected system" claim gets tested as a claim.** Showcase §11 argues that daily tracking
   makes the AI, the therapist and friends smarter. That is a cross-service assertion, so it gets
   real coverage ([E2E-M07](E2E-Mobile/E2E-M07-AI-Companion-Ban-Tam-Giao.md),
   [E2E-M13](E2E-Mobile/E2E-M13-Privacy-Data-Access-Grants.md)) rather than a demo hand-wave.

---

## 📁 What's in here

```
09-Testing/
├── README.md ............................... You are here (index + priority + status board)
├── 00-Test-Strategy-and-Conventions.md ..... Levels, test-case ID scheme, severity, entry/exit criteria
├── 01-Test-Environment-Builds-and-Data.md .. Devices, builds, accounts, backend URLs, data reset recipes
├── 02-Requirements-Traceability-Matrix.md .. Showcase feature ↔ test plan ↔ service coverage map
│
└── E2E-Mobile/ ............................. PHASE 1 — the teen mobile app, end to end
    ├── README.md ........................... Run order, smoke pack, full-regression pack
    ├── E2E-M01-Mood-Check-In-and-Focus-Mode.md
    ├── E2E-M02-Emotional-Journal.md
    ├── E2E-M03-Treasure-Box.md
    ├── E2E-M04-Sleep-and-Nutrition-Tracking.md
    ├── E2E-M05-Automatic-Step-Counting.md
    ├── E2E-M06-Breathing-and-Meditation.md
    ├── E2E-M07-AI-Companion-Ban-Tam-Giao.md
    ├── E2E-M08-Therapist-Matching-and-Booking.md
    ├── E2E-M09-Consultation-Session-and-Aftercare.md
    ├── E2E-M10-Social-Friends-and-Realtime-Chat.md
    ├── E2E-M11-Daily-Trophy-and-Streaks.md
    ├── E2E-M12-Crisis-Safety-and-Emergency-Support.md
    ├── E2E-M13-Privacy-Data-Access-Grants.md
    ├── E2E-M14-Notifications-End-to-End.md
    └── E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md
```

---

## 🗂️ Phase 1 — mobile E2E test plans

Ordered by execution priority. **P0** = the demo dies without it; **P1** = a showcased feature;
**P2** = supporting/polish.

| # | Test plan | Showcase § | Pri | Services exercised |
|---|---|---|---|---|
| M01 | [Mood check-in & Focus Mode reminders](E2E-Mobile/E2E-M01-Mood-Check-In-and-Focus-Mode.md) | §1 | **P0** | Auth, Tracking, Dashboard, Notifee (local) |
| M02 | [Emotional journal — Góc tâm tư](E2E-Mobile/E2E-M02-Emotional-Journal.md) | §2 | **P0** | Auth, Tracking, MinIO |
| M03 | [Treasure Box — Hộp kho báu](E2E-Mobile/E2E-M03-Treasure-Box.md) | §3 | **P0** | Auth, Tracking, MinIO |
| M04 | [Sleep & nutrition tracking](E2E-Mobile/E2E-M04-Sleep-and-Nutrition-Tracking.md) | §4 | **P1** | Auth, Tracking, Dashboard |
| M05 | [Automatic step counting](E2E-Mobile/E2E-M05-Automatic-Step-Counting.md) | §4 | **P1** | Tracking + device sensor |
| M06 | [Breathing & meditation](E2E-Mobile/E2E-M06-Breathing-and-Meditation.md) | §4 | **P1** | Tracking, YouTube/audio |
| M07 | [AI companion — Bạn Tâm Giao](E2E-Mobile/E2E-M07-AI-Companion-Ban-Tam-Giao.md) | §5, §11 | **P0** | Auth, AI, Tracking (grant), Gemini |
| M08 | [Therapist matching & booking](E2E-Mobile/E2E-M08-Therapist-Matching-and-Booking.md) | §6.1–6.2 | **P0** | Auth, Therapist, Notification |
| M09 | [Consultation session & aftercare](E2E-Mobile/E2E-M09-Consultation-Session-and-Aftercare.md) | §6.3–6.4 | **P0** | Therapist, Social, web UI |
| M10 | [Social friends & real-time chat](E2E-Mobile/E2E-M10-Social-Friends-and-Realtime-Chat.md) | §7 | **P1** | Auth, Social, RabbitMQ STOMP |
| M11 | [Daily trophy & streaks](E2E-Mobile/E2E-M11-Daily-Trophy-and-Streaks.md) | §8 | **P1** | Tracking, Dashboard |
| M12 | [Crisis safety & emergency support](E2E-Mobile/E2E-M12-Crisis-Safety-and-Emergency-Support.md) | §5, §9 | **P0** | AI (guardrail), device dialler |
| M13 | [Privacy — data-access grants](E2E-Mobile/E2E-M13-Privacy-Data-Access-Grants.md) | §10 | **P0** | Auth, Tracking, Social, web UI |
| M14 | [Notifications end-to-end](E2E-Mobile/E2E-M14-Notifications-End-to-End.md) | §6, §10 | **P1** | Therapist, Notification, FCM, SMTP |
| M15 | [First run, personalisation & session lifecycle](E2E-Mobile/E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md) | §10 | **P2** | Auth (refresh/logout), i18n, widget |

### Execution status board

Update this table as runs complete. `—` = not yet executed.

| Plan | Last run | Build | Result | Open defects |
|---|---|---|---|---|
| M01 | — | — | — | — |
| M02 | — | — | — | — |
| M03 | — | — | — | — |
| M04 | — | — | — | — |
| M05 | — | — | — | — |
| M06 | — | — | — | — |
| M07 | — | — | — | — |
| M08 | — | — | — | — |
| M09 | — | — | — | — |
| M10 | — | — | — | — |
| M11 | — | — | — | — |
| M12 | — | — | — | — |
| M13 | — | — | — | — |
| M14 | — | — | — | — |
| M15 | — | — | — | — |

---

## 🛣️ Later phases (not started)

Phase 1 is deliberately the whole focus. These are queued behind it, in the order we intend to
tackle them once the mobile E2E suite is green:

| Phase | Folder (planned) | Scope |
|---|---|---|
| **2** | `E2E-Web/` | Therapist web UI end-to-end: verification, schedule templates, patient list, session console, clinical notes. Partially covered today by `therapist-web-ui/docs/Manual_Test.md`, which should be absorbed here. |
| **3** | `Integration-API/` | Contract-level tests per service through the gateway (auth → tracking → dashboard fan-out, grant enforcement matrix, event publish/consume via RabbitMQ). |
| **4** | `Non-Functional/` | Performance (cold start, dashboard fan-out latency, chat round-trip), reliability (service-down degradation), and battery/data use for the background step counter. |
| **5** | `Security/` | JWT/JWKS verification matrix, grant-bypass attempts, IDOR on `profileId`-scoped endpoints, cleartext-traffic and secret-in-log checks. |
| **6** | `Accessibility-and-Compatibility/` | Font scaling, contrast, screen-reader labels, Android 10→15 matrix, small/large screens, VI/EN layout overflow. |
| **7** | `Automated/` | Jest component/hook tests for the mobile app and repair of the broken JVM suites (see the known-breakage note in §Known issues below). |

---

## ⚠️ Known issues every tester must read first

These are **pre-existing, documented conditions**, not bugs to re-discover. Raising them again as new
defects wastes a run.

1. **Seeded `*.dev@mhsa.local` accounts cannot log in.** The BCrypt hash in
   `V2__seed_development_testing_accounts.sql` does not verify against `password`. **Register fresh
   accounts** for every test run — see [01-Test-Environment](01-Test-Environment-Builds-and-Data.md).
2. **Only `appointment.booked` produces a notification.** Therapist `confirm` / `reject` / `cancel`
   publish no event, so they generate no inbox row, email or push. `streak.milestone` and
   `message.missed` consumers were deleted in July 2026 — those flows do not exist. Do not write
   "expected: push on therapist confirmation".
3. **The 43 seeded data-access grants exist only in `auth_db`.** Seed migrations INSERT directly and
   bypass the event pipeline, so Tracking's grant replica is empty until the nightly reconcile
   (23:30 ICT). **Create grants through the app**, never rely on seeded ones — especially before a demo.
4. **A released build older than 2026-07-20 has no AI consent toggle**, so its AI companion is
   permanently ungrounded (`[USER CONTEXT - NOT SHARED]`). Check the build under test before failing
   M07.
5. **JVM test suites are partly broken** (therapist-api does not compile its tests; thesis_social has
   2 deterministic failures and a ~50% flake). Irrelevant to Phase 1, but do not treat a red
   `./gradlew test` as a regression you caused.

---

*Maintained alongside the rest of the thesis documentation. Every plan here should stay true to the
implemented reality of the system — if a test plan and the code disagree, one of them is a defect.*
