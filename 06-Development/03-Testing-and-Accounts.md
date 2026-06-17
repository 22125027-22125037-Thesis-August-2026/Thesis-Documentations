# Testing & Test Accounts

> How to test uMatter: the seeded development accounts, manual test flows, and what automated tests
> exist. For running the stack, see [05-Deployment/03-Local-Development-Setup](../05-Deployment/03-Local-Development-Setup.md).

---

## 1. Seeded development & testing accounts

The Auth domain ships a deterministic seed migration
(`V2__seed_development_testing_accounts.sql`) that creates **90 accounts** so QA and demos have
predictable data:

| Role | Count | Email pattern | Password |
|---|---|---|---|
| `TEEN` | 30 | `teen001.dev@mhsa.local` … `teen030.dev@mhsa.local` | `developer` |
| `PARENT` | 30 | `parent001.dev@mhsa.local` … `parent030.dev@mhsa.local` | `developer` |
| `THERAPIST` | 30 | `therapist0NN.dev@mhsa.local` | `developer` |

- **Shared password for every account: `developer`.**
- IDs (`user_id`, `profile_id`) are **deterministic** — the same across rebuilds — so flows that need
  a known `profile_id` (grants, assignments, bookings) are reproducible.
- Parents are linked to teens via `linked_teen_user_id`; teens carry `school` + `emergency_contact`.

> **Full table** (every email, `user_id`, `profile_id`, school, links) is in the in-repo doc:
> `therapist-api/docs/development_and_testing_accounts.md` (mirrored in `thesis_social/docs/`).
> Example teen: `teen001.dev@mhsa.local` → profile_id `e1d0add5-b9c8-57b5-36e6-059991832f17`.

> ⚠️ These are **development** credentials in a seed migration for local/demo use. Don't seed them
> into a real production database.

---

## 2. Quick smoke test (through the gateway)

```bash
# 1. login as a seeded teen → grab the access token
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"teen001.dev@mhsa.local","password":"developer"}'

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
| **Social chat** | friend request → accept → real-time STOMP chat → recipient offline → `message.missed` → FCM push (Social, Notification) |
| **Notifications** | register device token → trigger an event → inbox row appears + push/email sent → mark read (Notification) |

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
