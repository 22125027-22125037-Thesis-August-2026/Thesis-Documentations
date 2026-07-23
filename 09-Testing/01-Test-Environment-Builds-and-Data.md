# 01 — Test Environment, Builds & Data

> Everything you need in place *before* opening a test plan. If a case says "Preconditions: standard
> teen account", this page defines what that means.

---

## 1. Systems under test

| Component | Under test | Endpoint / artefact |
|---|---|---|
| **Mobile app** (teen) | ✅ Phase 1 | `thesis-mobile` release APK/AAB, Android |
| **Therapist web UI** | Supporting role in Phase 1 (M09, M13, M14); own phase later | `https://umatter-apcs.duckdns.org` |
| **Backend (all services)** | Exercised through the clients | `https://umatter-apcs.duckdns.org` → Caddy → Nginx `:8080` → services |

### Topology you are testing through

```
Android device ──HTTPS──▶ Caddy (edge, Let's Encrypt, serves web UI, routes /ws)
                            │
                            ├──▶ Nginx :8080 (internal API gateway, /api/* path routing)
                            │       ├── auth-service        /api/v1/auth/*
                            │       ├── tracking-service    /api/v1/tracking/*
                            │       ├── ai-service          /api/v1/ai/*
                            │       ├── dashboard-service   /api/v1/dashboard/*
                            │       ├── therapist-api       /api/v1/therapist/*
                            │       ├── social-api          /api/v1/social/*
                            │       └── notification-api    /api/v1/notification/*
                            │
                            └──▶ social-api /ws  (STOMP over WebSocket → RabbitMQ relay)
```

Two consequences that matter while testing:

- **The app is HTTPS-only.** `BASE_URL` in [axiosClient.ts](../../thesis-mobile/src/api/axiosClient.ts)
  is `https://umatter-apcs.duckdns.org` in both dev and release, and cleartext traffic is blocked at
  the Android manifest level. You cannot point the app at a plain-HTTP local backend without
  rebuilding.
- **`/ws` bypasses Nginx.** Chat problems and REST problems have different failure surfaces; do not
  assume one proves the other.

---

## 2. Build under test

Record these for every run — a result without a build identity is not reusable.

| Field | How to get it |
|---|---|
| Mobile version | `app.json` / `package.json` version + `git rev-parse --short HEAD` in `thesis-mobile` |
| Build type | `npm run build:android` → **release** (this is what we test; debug builds behave differently for notifications and step permissions) |
| Backend commit | `git rev-parse --short HEAD` on the VM's deployment checkout |
| Migration state | `SELECT version, success FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5;` per DB |

### Build checks before you start

1. **`HARDCODED_TEST_TOKEN` must be empty** in `src/api/axiosClient.ts`. A non-empty value pins every
   request to one identity and silently invalidates every multi-account test in this folder.
2. **`google-services.json` is present** in `android/app/` — without it FCM token retrieval returns
   `null`, login still succeeds, and [M14](E2E-Mobile/E2E-M14-Notifications-End-to-End.md) will fail
   for an uninteresting reason.
3. **Release build, not debug**, for any notification, permission or performance case.

---

## 3. Devices

Minimum for Phase 1: **one physical Android device**. The emulator cannot do the step counter
(hardware step sensor) or realistic FCM/background behaviour.

| Slot | Device | Android | Used for |
|---|---|---|---|
| **Primary** | *(record the model)* | *(record)* | All plans |
| **Secondary** | *(any second Android device or a second account on the primary is not enough)* | — | M10 (two-party real-time chat), M13/M14 (grant + notification observed from the other side) |
| **Desktop browser** | Chrome/Edge | — | Therapist web UI for M09, M13, M14 |

> **M10 genuinely needs two devices.** Real-time chat between two accounts cannot be proven by logging
> out and back in on one handset — that tests message history, not delivery.

### Device state before a run

- Notifications **allowed** for uMatter (Android 13+ prompts at runtime).
- Physical activity / `ACTIVITY_RECOGNITION` **allowed** (needed by M05).
- Battery optimisation **disabled** for uMatter when testing scheduled reminders (M01) — aggressive
  OEM power management is the single most common cause of "the reminder never arrived", and it is a
  device behaviour, not a defect in the app.
- Device clock set to **automatic**, timezone **ICT (UTC+7)**. Several features key off the *local*
  calendar date (`toLocalDateKey` in `stepTracker.ts`, the dashboard's day boundary) — a manually
  skewed clock produces confusing, non-reproducible results.

---

## 4. Test accounts

### ⚠️ Do not use the seeded accounts

`V2__seed_development_testing_accounts.sql` creates 90 accounts (`teen001.dev@mhsa.local`, etc.) whose
BCrypt hash does not verify against `password` or any known plaintext. **They cannot log in.** Their
only remaining use is as *deterministic profile IDs* referenced by other seed data.

Likewise the **43 seeded data-access grants exist only in `auth_db`** — the seed INSERTs bypass the
event pipeline, so Tracking's replica does not have them until the nightly reconcile (23:30 ICT).
Never build a test (or a demo) on a seeded grant. Create grants through the app.

### Accounts to create for each run

Register fresh through the app or `POST /api/v1/auth/register`. Use `+qa` addressing on a mailbox you
control so [M14](E2E-Mobile/E2E-M14-Notifications-End-to-End.md) can verify the confirmation email.

| Alias | Role | Purpose | Suggested email |
|---|---|---|---|
| **TEEN-A** | TEEN | The main subject. All tracking, AI, booking cases. | `you+qa-teenA-<date>@gmail.com` |
| **TEEN-B** | TEEN | Friend/counterparty for M10 and M13 (peer grant). | `you+qa-teenB-<date>@gmail.com` |
| **THERAPIST-T** | THERAPIST | Counterparty for M08/M09/M13/M14 on the web UI. Needs verification + a published schedule. | `you+qa-ther-<date>@gmail.com` |

Record, for each: **email, password, `profileId`** (from `GET /api/v1/auth/me`), and which device it
is signed in on.

```bash
# Register → login → capture the identity you will paste into every backend-verification step
curl -s -X POST https://umatter-apcs.duckdns.org/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"you+qa-teenA-20260723@gmail.com","password":"<pw>","fullName":"QA Teen A","role":"TEEN"}'

TOKEN=$(curl -s -X POST https://umatter-apcs.duckdns.org/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"you+qa-teenA-20260723@gmail.com","password":"<pw>"}' | jq -r .data.accessToken)

curl -s https://umatter-apcs.duckdns.org/api/v1/auth/me -H "Authorization: Bearer $TOKEN"
# → note profileId
```

> **Keep the demo account separate.** Destructive cases (revoke grant, cancel appointment, unfriend,
> delete diary entry) run on QA accounts only.

### Therapist setup (needed before M08)

A newly registered therapist cannot be booked. Before running M08:

1. Sign in to the web UI as THERAPIST-T and complete the profile (specialties, bio, languages).
2. Have the account marked **verified** (admin action or direct DB update — record which you did).
3. Create a **schedule template** covering the next few days.
4. Materialise slots without waiting for the cron:
   ```bash
   curl -X POST https://umatter-apcs.duckdns.org/api/v1/test/trigger-generation
   ```
   (Permit-all test endpoint on therapist-api. `trigger-cleanup` deletes stale unbooked slots.)
5. Confirm from the mobile side that slots appear:
   `GET /api/v1/therapist/therapists/{id}/available-slots`.

---

## 5. Seeding realistic tracking history

Several plans need *history*, not a single row: the 7-day trends (M04), the mood calendar (M02), a
streak (M11), and — most importantly — a grounded AI reply (M07). A brand-new account has none.

Two options:

**A. Through the app (slower, more faithful).** Log entries for several past days using each screen's
date picker. This is what the plans assume by default, and it is what catches date-boundary bugs.

**B. Through the API (fast setup for cases that are not *about* the write path).**

```bash
# a mood a day for the last 7 days
for i in $(seq 1 7); do
  curl -s -X POST https://umatter-apcs.duckdns.org/api/v1/tracking/moods/ \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d "{\"moodTag\":\"GOOD\",\"positivityScore\":8,\"content\":\"QA seed day $i\"}"
done
```

Adjust the payload per resource — the collection write paths are `POST /api/v1/tracking/{moods,
sleeps, foods, steps, breathing, diaries, treasures}/` (note the trailing slash) and reads are
`GET /api/v1/tracking/{resource}/{profileId}`.

> If a case's own subject is the write path (M01-01, M02-01, …) you **must** use option A. Option B is
> for arranging the *background* a later case needs.

---

## 6. Resetting between runs

There is no "reset test data" endpoint. In descending order of preference:

1. **Register a new account.** Cheapest, cleanest, and what these plans assume. The cost is
   re-doing therapist assignment/grants for that account.
2. **Delete rows through the app** (diary entries, treasures, logs all have delete affordances) —
   also exercises the delete paths.
3. **Delete rows via API** — `DELETE /api/v1/tracking/{resource}/{id}` with the owner's token.
4. **Direct SQL** — last resort, and record it in the run log, because it can leave the Tracking
   grant replica or Dashboard cache inconsistent in ways a real user could never produce.

App-side state that survives logout and can poison a rerun:

| Key (AsyncStorage) | What it holds | Clear by |
|---|---|---|
| `@low_mood_prompt_last_shown` | 6-hour cooldown on the low-mood safety-net sheet | Reinstall, or wait 6 h (M01-05 depends on it) |
| `@focus_mode_enabled`, `@focus_mode_notification_ids`, `@focus_mode_quiet_hours` | Focus Mode schedule | Toggle Focus Mode off, or reinstall |
| `@steps/daily_baseline` | Cumulative sensor baseline for today | Reinstall (M05-04 depends on it) |
| Tour "seen" flag | Whether the first-run coach-marks auto-start | Replay from Profile, or reinstall (M15-01) |

**When a case's precondition is "first-run state", it means a clean install** — clearing app data is
equivalent; logging out is not.

---

## 7. Backend verification cheat-sheet

Paths as the mobile client calls them (from `src/api/*.ts`), for pasting into `curl`:

| Domain | Read | Write |
|---|---|---|
| Mood | `GET /api/v1/tracking/moods/{profileId}` | `POST /api/v1/tracking/moods/` |
| Sleep | `GET /api/v1/tracking/sleeps/{profileId}` | `POST /api/v1/tracking/sleeps/` |
| Food | `GET /api/v1/tracking/foods/{profileId}` | `POST /api/v1/tracking/foods/` |
| Steps | `GET /api/v1/tracking/steps/{profileId}` · `…/range?from=&to=` | `POST /api/v1/tracking/steps/` |
| Breathing | `GET /api/v1/tracking/breathing/{profileId}` · `…/range` | `POST /api/v1/tracking/breathing/` |
| Diary | `GET /api/v1/tracking/diaries/{profileId}` | `POST /api/v1/tracking/diaries/` |
| Treasure | `GET /api/v1/tracking/treasures/` | `POST /api/v1/tracking/treasures/` |
| Streak | `GET /tracking/streaks` | — |
| Dashboard | `GET /api/v1/dashboard/summary` | — |
| AI chat | `GET /api/v1/ai/chat/sessions` · `…/history/{sessionId}` | `POST /api/v1/ai/chat/send` |
| Grants | `GET /api/v1/auth/grants/{profileId}` · `…/{profileId}/received` · `…/status/{otherId}` | `POST /api/v1/auth/grants` · `DELETE /api/v1/auth/grants/{granteeProfileId}` |
| Friends | `GET /api/v1/social/friends/requests` | `POST /api/v1/social/friends/requests` · `…/{id}/accept` · `…/{id}/reject` · `DELETE /api/v1/social/friends/{profileId}` |
| Chat | `GET /api/v1/social/chats/channels` · `…/channels/{id}/messages` | STOMP `/app/chat.send` → `/user/queue/messages` |
| Therapist | `GET /api/v1/therapist/therapists` · `…/{id}` · `…/{id}/available-slots` · `…/appointments/{id}` | `POST …/bookings` · `…/appointments/{id}/cancel` · `POST …/matching/preferences` · `POST …/reviews` |
| Notifications | `GET /api/v1/notification/notifications` · `…/unread-count` | `PUT …/{id}/read` · `PUT …/read-all` |

**The AI companion's reserved grantee id** (for M07/M13):
`a1000000-a1a1-4a1a-8a1a-a1a1a1a1a1a1`.

---

## 8. Diagnosing, not guessing

Three probes that repeatedly save a run:

**Is the app actually talking to the backend?**
```bash
adb logcat -s ReactNativeJS:V | grep -E "API|axios|status"
```

**Is STOMP connected (M10)?** The upgrade succeeding proves nothing — the historical chat bug lived
one layer above it. Look for the app's own log line:
```
STOMP connected successfully { brokerURL: wss://umatter-apcs.duckdns.org/ws }
```
and, server-side, `WebSocketMessageBrokerStats` reporting a non-zero session count in social-api's
logs. When curling the endpoint by hand, force HTTP/1.1 (`curl --http1.1`) — HTTP/2 forbids `Upgrade`
headers and returns a misleading 400 on a healthy endpoint.

**Did the notification event fire (M14)?** Check the RabbitMQ management UI for `appointment.booked`
and notification-api's consumer log. Remember: it is the **only** live producer.
