# 02 — Requirements Traceability Matrix

> Every showcased feature maps to at least one test case. This page is the proof — and the place where
> a coverage gap becomes visible before the council finds it.

Source of the "requirement" column: [08-Product-Showcase/Mobile-App-Showcase](../08-Product-Showcase/Mobile-App-Showcase.md).
That document is what the app *claims to do*, so it is the specification Phase 1 tests against.

---

## 1. Showcase feature → test coverage

| Showcase § | Feature claimed | Test plan | Key cases | Pri |
|---|---|---|---|---|
| §1 | Daily mood check-in, 5 moods, note | [M01](E2E-Mobile/E2E-M01-Mood-Check-In-and-Focus-Mode.md) | M01-01, M01-02, M01-03 | P0 |
| §1 | Focus Mode — nudge ≈ every 90 min | [M01](E2E-Mobile/E2E-M01-Mood-Check-In-and-Focus-Mode.md) | M01-06, M01-07 | P1 |
| §1 | Quiet Hours 22:00–07:00 respected | [M01](E2E-Mobile/E2E-M01-Mood-Check-In-and-Focus-Mode.md) | M01-08 | P1 |
| §1 | Reminders "quietly refill themselves" | [M01](E2E-Mobile/E2E-M01-Mood-Check-In-and-Focus-Mode.md) | M01-09 | P2 |
| §1 | Low mood → caring response / path to support | [M01](E2E-Mobile/E2E-M01-Mood-Check-In-and-Focus-Mode.md) · [M12](E2E-Mobile/E2E-M12-Crisis-Safety-and-Emergency-Support.md) | M01-05, M12-05 | P0 |
| §2 | Journal entry: title, mood, body, ≤5 photos | [M02](E2E-Mobile/E2E-M02-Emotional-Journal.md) | M02-01, M02-02, M02-03 | P0 |
| §2 | Mood calendar colouring pos/neu/neg | [M02](E2E-Mobile/E2E-M02-Emotional-Journal.md) | M02-06 | P1 |
| §2 | Journal streak (🔥 X ngày) | [M02](E2E-Mobile/E2E-M02-Emotional-Journal.md) · [M11](E2E-Mobile/E2E-M11-Daily-Trophy-and-Streaks.md) | M02-07, M11-04 | P1 |
| §2 | Search / filter by date | [M02](E2E-Mobile/E2E-M02-Emotional-Journal.md) | M02-08 | P2 |
| §3 | Treasure Box — save & reopen good memories | [M03](E2E-Mobile/E2E-M03-Treasure-Box.md) | M03-01 … M03-05 | P0 |
| §4 | Sleep log: bedtime, wake, 5 quality levels | [M04](E2E-Mobile/E2E-M04-Sleep-and-Nutrition-Tracking.md) | M04-01, M04-02 | P1 |
| §4 | Nutrition: meals, water (L), fullness, mindful eating | [M04](E2E-Mobile/E2E-M04-Sleep-and-Nutrition-Tracking.md) | M04-04, M04-05 | P1 |
| §4 | 7-day trends behind each card | [M04](E2E-Mobile/E2E-M04-Sleep-and-Nutrition-Tracking.md) | M04-07 | P1 |
| §4 | Automatic background step counting | [M05](E2E-Mobile/E2E-M05-Automatic-Step-Counting.md) | M05-01 … M05-05 | P1 |
| §4 | Daily goal + "Đã đạt mục tiêu! 🎉" | [M05](E2E-Mobile/E2E-M05-Automatic-Step-Counting.md) | M05-06 | P2 |
| §4 | Guided breathing (inhale/hold/exhale) + music | [M06](E2E-Mobile/E2E-M06-Breathing-and-Meditation.md) | M06-01 … M06-04 | P1 |
| §4 | Meditation carousel (5 sessions) | [M06](E2E-Mobile/E2E-M06-Breathing-and-Meditation.md) | M06-06, M06-07 | P1 |
| §5 | AI companion available 24/7, warm persona | [M07](E2E-Mobile/E2E-M07-AI-Companion-Ban-Tam-Giao.md) | M07-01, M07-02 | P0 |
| §5 | Quick-reply chips | [M07](E2E-Mobile/E2E-M07-AI-Companion-Ban-Tam-Giao.md) | M07-03 | P2 |
| §5, §11 | **Grounded in the user's own tracking context** | [M07](E2E-Mobile/E2E-M07-AI-Companion-Ban-Tam-Giao.md) | M07-06, M07-07 | P0 |
| §5 | Crisis → hotline surfaced immediately | [M12](E2E-Mobile/E2E-M12-Crisis-Safety-and-Emergency-Support.md) | M12-01, M12-02 | P0 |
| §5 | Session history / resume a past chat | [M07](E2E-Mobile/E2E-M07-AI-Companion-Ban-Tam-Giao.md) | M07-04, M07-05 | P1 |
| §6.1 | 8-step matching intake → compatible therapist | [M08](E2E-Mobile/E2E-M08-Therapist-Matching-and-Booking.md) | M08-01 … M08-04 | P0 |
| §6.2 | Book a free slot in a tap | [M08](E2E-Mobile/E2E-M08-Therapist-Matching-and-Booking.md) | M08-06, M08-07 | P0 |
| §6, §10 | Booking → push **and** email confirmation | [M14](E2E-Mobile/E2E-M14-Notifications-End-to-End.md) | M14-02, M14-03, M14-04 | P1 |
| §6 | Appointment control: view, confirm, cancel w/ reason | [M08](E2E-Mobile/E2E-M08-Therapist-Matching-and-Booking.md) | M08-09, M08-10 | P1 |
| §6.3 | Join by video or chat from the app | [M09](E2E-Mobile/E2E-M09-Consultation-Session-and-Aftercare.md) | M09-02 … M09-05 | P0 |
| §6.4 | Read therapist's notes after the session | [M09](E2E-Mobile/E2E-M09-Consultation-Session-and-Aftercare.md) | M09-07 | P1 |
| §6.4 | Leave rating & review; therapist rating updates | [M09](E2E-Mobile/E2E-M09-Consultation-Session-and-Aftercare.md) | M09-08, M09-09 | P1 |
| §7 | Add friends by email, accept/decline | [M10](E2E-Mobile/E2E-M10-Social-Friends-and-Realtime-Chat.md) | M10-01 … M10-04 | P1 |
| §7 | Real-time chat between friends | [M10](E2E-Mobile/E2E-M10-Social-Friends-and-Realtime-Chat.md) | M10-05 … M10-08 | P1 |
| §8 | Daily trophy: bronze / silver / gold | [M11](E2E-Mobile/E2E-M11-Daily-Trophy-and-Streaks.md) | M11-01, M11-02, M11-03 | P1 |
| §8 | Profile shows streak, AI sessions, member since | [M11](E2E-Mobile/E2E-M11-Daily-Trophy-and-Streaks.md) | M11-05 | P2 |
| §9 | Emergency support always one tap away | [M12](E2E-Mobile/E2E-M12-Crisis-Safety-and-Emergency-Support.md) | M12-06 … M12-08 | P0 |
| §10 | Therapist sees data **only** with permission | [M13](E2E-Mobile/E2E-M13-Privacy-Data-Access-Grants.md) | M13-01 … M13-05 | P0 |
| §10 | Revoke at any time | [M13](E2E-Mobile/E2E-M13-Privacy-Data-Access-Grants.md) | M13-06, M13-07 | P0 |
| §10 | Per-category scope (not all-or-nothing) | [M13](E2E-Mobile/E2E-M13-Privacy-Data-Access-Grants.md) | M13-03, M13-04 | P0 |
| §10 | Bilingual VI / EN | [M15](E2E-Mobile/E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md) | M15-04, M15-05 | P2 |
| §10 | Light / dark / system theme | [M15](E2E-Mobile/E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md) | M15-06 | P2 |
| §10 | First-run tour, replayable | [M15](E2E-Mobile/E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md) | M15-01, M15-02 | P2 |
| §10 | Home-screen widget | [M15](E2E-Mobile/E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md) | M15-07 | P2 |
| §11 | Tracking makes AI + therapist + friend smarter | [M07](E2E-Mobile/E2E-M07-AI-Companion-Ban-Tam-Giao.md) · [M13](E2E-Mobile/E2E-M13-Privacy-Data-Access-Grants.md) | M07-06, M13-04, M13-09 | P0 |

---

## 2. Service coverage — which plan exercises which service

|  | Auth | Tracking | AI | Dashboard | Therapist | Social | Notification | MinIO | RabbitMQ |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| M01 | ● | ● | | ● | | | | | |
| M02 | ● | ● | | | | | | ● | |
| M03 | ● | ● | | | | | | ● | |
| M04 | ● | ● | | ● | | | | | |
| M05 | ● | ● | | ● | | | | | |
| M06 | ● | ● | | | | | | | |
| M07 | ● | ● | ● | | | | | | |
| M08 | ● | | | | ● | | ● | | ● |
| M09 | ● | | | | ● | ● | | | |
| M10 | ● | | | | | ● | | | ● |
| M11 | ● | ● | | ● | | | | | |
| M12 | ● | | ● | | | | | | |
| M13 | ● | ● | ● | | ● | ● | | | ● |
| M14 | ● | | | | ● | | ● | | ● |
| M15 | ● | ● | | ● | | | | | |

**Reading the gaps:** Notification-API is only reachable through M08/M14 because
`appointment.booked` is the only live producer — that is a property of the system, not a hole in the
plan. MinIO appears only where media upload exists (journal photos, treasure media, avatars).

---

## 3. Cross-client workflows

Three workflows cannot be proven from the phone alone. They need the therapist web UI open in a
browser at the same time, and they are where the seams of the architecture actually show:

| Workflow | Mobile side | Web side | Plan |
|---|---|---|---|
| Booking → session → notes → review | Book, join, read note, rate | Accept, run session, write clinical note, see rating change | [M09](E2E-Mobile/E2E-M09-Consultation-Session-and-Aftercare.md) |
| Consent lifecycle | Grant, scope, revoke | Read patient data before/after each change | [M13](E2E-Mobile/E2E-M13-Privacy-Data-Access-Grants.md) |
| Booking notification fan-out | Receive push, open inbox row | Trigger the booked appointment's counterpart state | [M14](E2E-Mobile/E2E-M14-Notifications-End-to-End.md) |

---

## 4. Deliberately untested (and why)

Recording these keeps "we didn't test it" separate from "we forgot".

| Area | Why it is out of scope for Phase 1 |
|---|---|
| Therapist web UI in its own right | Phase 2. Currently covered by `therapist-web-ui/docs/Manual_Test.md`. |
| Parent / admin experiences | Screens exist in the navigator (`ParentExperience`, `AdminExperience`) but no parent→teen link exists at schema level; not a showcased feature. |
| iOS build | No iOS release is part of the deliverable. |
| Gemini answer quality | Third-party, non-deterministic. We test grounding *inputs* and safety *outputs*, not prose quality. |
| `streak.milestone` / `message.missed` notifications | **The consumers were deleted.** These flows do not exist; testing them would produce a false defect. |
| Therapist confirm/reject/cancel notifications | No event is published by those actions. Documented gap, not a test target. |
| Load / concurrency | Phase 4. A thesis demo is single-user; capacity claims are not being made. |

---

## 5. How to keep this page honest

When a test plan gains or loses a case, update the row here in the same edit. A traceability matrix
that drifts from its plans is worse than none — it manufactures confidence. The check is cheap: every
plan's case IDs should appear here, and every showcase section should have at least one **P0 or P1**
row.
