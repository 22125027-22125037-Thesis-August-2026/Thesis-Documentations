# Testing & Test Accounts

> How to test uMatter: the seeded development accounts, manual test flows, and what automated tests
> exist. For running the stack, see [05-Deployment/03-Local-Development-Setup](../05-Deployment/03-Local-Development-Setup.md).

---

## 1. Seeded development & testing accounts

The Auth domain ships a deterministic seed migration
(`V2__seed_development_testing_accounts.sql`) that creates **90 accounts** so QA and demos have
predictable data:

| Role | Count | Email pattern |
|---|---|---|
| `TEEN` | 30 | `teen001.dev@mhsa.local` … `teen030.dev@mhsa.local` |
| `PARENT` | 30 | `parent001.dev@mhsa.local` … `parent030.dev@mhsa.local` |
| `THERAPIST` | 30 | `therapist0NN.dev@mhsa.local` |

- ⚠️ **The seeded accounts cannot currently log in.** The seed's comment documents the plaintext as
  `password`, but the BCrypt hash it inserts does **not** verify against that (or any known)
  plaintext — confirmed during the 2026-07-17 V6 deploy verification, on prod too. Until the seed
  hash is regenerated, **register fresh accounts for manual testing** (or reset a seeded account's
  password hash by hand in the DB).
- `profile_id`s are **deterministic** (md5-derived) — the same across rebuilds — so flows that need
  a known `profile_id` (grants, assignments, bookings) are reproducible. (Since the V6
  users→profiles merge, `profile_id` is the only id; seeded `user_id`s are gone.)
- There is **no schema-level parent→teen link** (no `linked_teen_user_id` column exists); a parent
  account is just `role = PARENT`, and any parent↔teen relationship is made in the app the same way
  as a friendship, plus an optional data-access grant.

> **Full table** (every email, `profile_id`, school) is in the in-repo doc:
> `therapist-api/docs/development_and_testing_accounts.md` (mirrored in `thesis_social/docs/`).
> Example teen: `teen001.dev@mhsa.local` → profile_id `e1d0add5-b9c8-57b5-36e6-059991832f17`.

> ⚠️ These are **development** credentials in a seed migration for local/demo use. Don't seed them
> into a real production database.

---

## 2. Quick smoke test (through the gateway)

> ⚠️ Step 1 fails against the broken seed hash (§1) — register a fresh account first and log in
> with that instead.

```bash
# 1. login → grab the access token (see warning above re: seeded credentials)
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"teen001.dev@mhsa.local","password":"password"}'

TOKEN=<accessToken from above>

# 2. who am I?
curl -s http://localhost:8080/api/v1/auth/me -H "Authorization: Bearer $TOKEN"

# 3. log a mood
curl -s -X POST http://localhost:8080/api/v1/tracking/moods/ \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"moodScore":7,"notes":"Feeling better","emotionTags":["happy"]}'

# 4. talk to the AI companion
curl -s -X POST http://localhost:8080/api/v1/ai/chat/send \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"message":"I had a rough day"}'

# 5. dashboard summary (exercises the BFF → auth+tracking+ai)
curl -s http://localhost:8080/api/v1/dashboard/summary -H "Authorization: Bearer $TOKEN"
```

---

## 3. End-to-end manual flows worth testing

| Flow | Steps (services touched) |
|---|---|
| **Self-care loop** | login → log mood/sleep/diary → view streaks → AI chat (Auth, Tracking, AI) |
| **Matching + booking** | submit matching intake → get matches → auto-assign → list slots → book → receive confirmation email/inbox (Therapist, Notification) |
| **Video session** | join within the 10-min window → `IN_PROGRESS` → therapist writes note → `COMPLETED` → patient leaves a review → therapist rating updates (Therapist) |
| **Consent + therapist view** | patient grants access → therapist reads patient tracking context; revoke → access denied (Auth, Tracking) |
| **Social chat** | friend request → accept → real-time STOMP chat, both clients online (Social). ⚠️ **Stop there** — the "recipient offline → `message.missed` → FCM push" step **cannot be tested: Social never publishes that event.** |
| **Notifications** | register device token → **book an appointment** (the only live producer) → inbox row appears + confirmation email **and** push sent → mark read (Therapist, Notification) |

The therapist web UI has a detailed manual test plan in `therapist-web-ui/docs/Manual_Test.md`.

---

## 4. Triggering scheduled jobs on demand

The Therapist API exposes permit-all test endpoints so you don't have to wait for the cron:
```bash
curl -X POST http://localhost:8085/api/v1/test/trigger-generation   # materialise slots from templates
curl -X POST http://localhost:8085/api/v1/test/trigger-cleanup      # delete stale unbooked slots
```

---

## 5. Automated tests

| Service | Coverage (representative) |
|---|---|
| **Therapist API** | `ScheduleGenerationServiceIntegrationTest` (TZ conversion, idempotent generation), `ReviewServiceIntegrationTest` (creation, duplicate prevention, non-completed rejection, rating recompute), context-startup test |
| Others | Spring context-load tests; mobile app has Jest config (`jest.config.js`) |

Run the Gradle repos' tests with `./gradlew test`; the Maven monorepo with `./mvnw test`.

> Test coverage is an explicit area for future expansion — see
> [07-Academic](../07-Academic/01-Thesis-Context-and-Future-Work.md).
