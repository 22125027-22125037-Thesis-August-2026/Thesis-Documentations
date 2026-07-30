# Testing & Test Accounts

> How to test uMatter: the seeded development accounts, manual test flows, and what automated tests
> exist. For running the stack, see [05-Deployment/03-Local-Development-Setup](../05-Deployment/03-Local-Development-Setup.md).
>
> 📋 **This page is the developer's quick reference — how to poke the system.** The *deliberate*,
> written-down test suite lives in **[09-Testing](../09-Testing/README.md)**: 15 detailed end-to-end
> mobile test plans with numbered cases, backend verification steps and run logs. Start there for a
> QA run, a demo rehearsal, or a pre-defence check. The §3 table below is the short version of what
> [09-Testing/E2E-Mobile](../09-Testing/E2E-Mobile/README.md) covers case by case.

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

- ✅ **The seeded accounts log in — the password is `developer`, not `password`.** Verified against
  prod 2026-07-30 (`teen001`, `teen008`, `teen021`, `parent001` all returned 200; `password` returns
  401). The seed file's own comment (`-- Plaintext password: password`) is what misled three earlier
  documentation passes into recording these accounts as unusable: the BCrypt hash is perfectly
  valid — `$2a$10$r0BNJBQr0qlwm…`, byte-identical in prod to what V2 inserts — it just encodes
  `developer`.
  > ⚠️ **Do not "correct" the comment in `V2__seed_development_testing_accounts.sql`.** Flyway
  > checksums migration files, so editing an already-applied migration makes auth-service fail
  > validation and crash-loop on its next boot. The real password is documented here instead.
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

```bash
# 1. login → grab the access token (seeded password is `developer`, see §1)
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
| **Video session** | join within the 10-min window → `IN_PROGRESS` → therapist writes note → `PROFESSIONAL_COMPLETE` → patient leaves a review → `OVERALL_COMPLETE` + therapist rating updates (Therapist). Reviewing first flips the order (`PATIENT_COMPLETE`, then the therapist has 24h to still file the note) |
| **Consent + therapist view** | patient grants access → therapist reads patient tracking context; revoke → access denied (Auth, Tracking) |
| **Social chat** | friend request → accept → real-time STOMP chat, both clients online (Social). ⚠️ **Stop there** — there is no "recipient offline → push" step. Social never published `message.missed`, and the consumer that suggested otherwise was removed in July 2026. |
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

| Service | Coverage (representative) | State (re-measured 2026-07-30) |
|---|---|---|
| **Therapist API** | `BookingServiceTest`, `ClinicalNoteServiceTest`, `VideoProviderSelectionTest` (Mockito / context-runner, no DB); `ScheduleGenerationServiceIntegrationTest` (TZ conversion, idempotent generation), `ReviewServiceIntegrationTest` (creation, duplicate prevention, non-completed rejection, rating recompute), context-startup test | 🟢 **46 / 46 green**, stable across three consecutive `--rerun-tasks` runs. The integration tests moved from H2 to a **PostgreSQL Testcontainer** running the real Flyway migrations (`ddl-auto: validate`), which removed the array-column flake entirely |
| **Social** | 47 tests incl. `JwtTokenServiceTest` (14, green), `ChatServiceTest`, `FriendServiceTest` | 🟡 **44 pass / 1 fail / 2 skipped** — the single failure is the `FriendServiceTest` strict-stubbing flake below |
| **Notification** | consumer + dispatcher tests | 🟢 green |
| Others | Spring context-load tests; mobile app has Jest config (`jest.config.js`) | — |

Run the Gradle repos' tests with `./gradlew test`. The Maven monorepo's wrapper was **repaired on
2026-07-30** (`.mvn/wrapper/maven-wrapper.properties` had never been committed, so `./mvnw` could not
resolve a distribution); `mvnw.cmd -v` and a real goal both succeed now. On this laptop use
`mvnw.cmd` from PowerShell — under Git Bash the wrapper's `wget` hits an msys2/cygwin DLL clash that
has nothing to do with Maven.

> ⚠️ **therapist-api's integration tests now need a running Docker daemon**, because they start a
> PostgreSQL Testcontainer. Without Docker they fail at startup rather than falling back to H2.

### Known test-suite problems (don't mistake these for your own breakage)

1. ~~**therapist-api's H2 array-column flake**~~ — **fixed 2026-07-30.** History, because the failure
   mode is worth recognising elsewhere: entities with `varchar[]` columns
   (`therapists.treated_challenges`, `patient_tags.tags`, `profiles_preferences.reasons`) generated
   DDL that H2's `MODE=PostgreSQL` rejects. Hibernate's `create-drop` logged each failed
   `CREATE TABLE` as a WARN and **kept going**, so which *other* tables ended up missing varied run
   to run and the symptom was a misleading `Table "THERAPISTS" not found` on an unrelated INSERT.
   The two `@SpringBootTest` classes now extend `AbstractPostgresIntegrationTest`, which starts a
   single shared **PostgreSQL Testcontainer** and runs the **real Flyway migrations** against it, so
   they exercise the deployed schema under `ddl-auto: validate` exactly as production does.
   *(The older "the tests do not compile" problem was fixed earlier, by `1c1a3cc`.)*
   > If you reuse this base class: it deliberately avoids `@Testcontainers`/`@Container`, whose
   > extension stops the container **per test class** — the first class passes and every later one
   > fails with `Connection to localhost:<port> refused`. A static singleton started once is correct.
2. ~~**thesis_social's 2 deterministic `ChatServiceTest` failures**~~ — **fixed 2026-07-30**, and they
   did *not* share a cause, though this page previously said they did:
   - `sendMessageShouldPersistNotifyAndPublishEvent` verified a `message_sent` publish that is
     commented out in `ChatService` and whose consumer was deleted in July 2026. The assertion was
     **dropped** — there is no such event anywhere in the system to assert on.
   - `listChannelsShouldIncludeCounterpartLastMessageAndUnreadCount` was **a defect in the test, not
     a dormant feature**: `listChannels` resolves names through the batch
     `resolveProfileNames(Set<UUID>)`, which the test never stubbed, so `counterpartUsername` came
     back `null`. Fixed by adding the missing stub, keeping the assertion and its coverage.
3. **`FriendServiceTest.unfriendShouldDeleteFriendshipAndDirectChannel` is flaky (~50%)**, failing
   with Mockito's `PotentialStubbingProblem` only in a full-suite run and passing in isolation —
   a strict-stubbing order dependence. Measured at 2/4 on both `main` and a feature branch, so a
   single red run is **not** evidence that a change broke it. Compare failure *rates*, not one run.

> Test coverage is an explicit area for future expansion — see
> [07-Academic](../07-Academic/01-Thesis-Context-and-Future-Work.md).
