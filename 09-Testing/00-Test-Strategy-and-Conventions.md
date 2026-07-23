# 00 — Test Strategy & Conventions

> Read this once. Everything in [09-Testing](README.md) follows the conventions defined here.

---

## 1. What we are testing, and why in this order

uMatter is a **microservice system with a mobile client**. The risk is not concentrated in any one
service — it is concentrated in the **seams**: an event that never fires, a grant replica that never
syncs, a STOMP frame that never connects. Unit tests do not see seams. So the strategy is
**outside-in**:

```
        ┌──────────────────────────────────────────────┐
Phase 1 │  E2E on a real Android device                 │  ← highest value, where we start
        │  real build → Caddy → Nginx → services → DB   │
        └──────────────────────────────────────────────┘
        ┌──────────────────────────────────────────────┐
Phase 3 │  API / contract tests through the gateway     │
        └──────────────────────────────────────────────┘
        ┌──────────────────────────────────────────────┐
Phase 7 │  Unit / component tests inside each repo      │  ← cheapest, last
        └──────────────────────────────────────────────┘
```

This inverts the classic test pyramid on purpose. The pyramid optimises for *cost of maintenance over
years*; a thesis optimises for *demonstrable correctness of the whole product on a fixed date*. Two
different objectives, two different shapes.

### The explicit non-goals

We do **not** write test plans for:

- Single-field validation UI (email format, password length, required-field asterisks) — these are
  exercised implicitly inside the workflows that use them, and only asserted where failure blocks the
  user. A test case whose expected result is "the red text appears" is not worth a numbered ID.
- Static content screens (About, FAQ, Contact, Terms) beyond "opens, renders, back works" — covered
  in one case in [M15](E2E-Mobile/E2E-M15-First-Run-Personalisation-and-Session-Lifecycle.md).
- Styling and pixel positions, except where a layout failure hides a control (checked as part of
  the compatibility phase).
- Third-party behaviour we do not control (Gemini's exact wording, YouTube playback quality, the
  Android dialler). We test *our* handling of them.

---

## 2. Test levels and where each lives

| Level | Definition for this project | Where |
|---|---|---|
| **E2E (system) test** | A real user goal completed on a real device against the deployed stack, with backend state verified afterwards. | `E2E-Mobile/`, `E2E-Web/` |
| **Cross-client E2E** | A workflow that spans the mobile app *and* the therapist web UI (booking → session → note → review). | `E2E-Mobile/E2E-M09`, `E2E-M13` |
| **Integration / contract test** | HTTP-level assertions on one service or one gateway path, no UI. | `Integration-API/` (Phase 3) |
| **Non-functional test** | Latency, resource use, degradation, security posture. | `Non-Functional/`, `Security/` |
| **Unit / component test** | Inside a repository, no network. | Each repo's own suite |

---

## 3. Test-case ID scheme

```
E2E-M07-04
│   │   └── case number, sequential within the plan, never reused
│   └────── plan number (M = mobile, W = web, I = integration, N = non-functional, S = security)
└────────── level
```

- IDs are **stable**. If a case is deleted, its number is retired, not recycled — an old defect report
  that references `E2E-M07-04` must never resolve to a different case later.
- Sub-cases use a letter suffix when a case has genuine variants that share every step but one:
  `E2E-M04-03a` (sleep), `E2E-M04-03b` (nutrition).

---

## 4. Priority levels

| Pri | Meaning | Rule of thumb |
|---|---|---|
| **P0** | Blocks the demo or breaks a headline claim of the thesis. | If this fails, we do not show the app. |
| **P1** | A showcased feature degrades but the demo survives. | Fix before the defence. |
| **P2** | Polish, edge case, or convenience. | Fix if time allows; document otherwise. |

Every test case carries a priority. A plan's own priority is the **highest** priority among its cases.

---

## 5. Severity of defects found

| Sev | Definition | Example |
|---|---|---|
| **S1 — Critical** | Data loss, crash on a main path, security/privacy breach, or a P0 workflow that cannot complete. | Revoking a grant still lets the therapist read the journal. |
| **S2 — Major** | A showcased feature is wrong or unusable, but a workaround exists. | Focus Mode schedules reminders inside quiet hours. |
| **S3 — Minor** | Incorrect but non-blocking behaviour. | Streak count off by one after midnight. |
| **S4 — Cosmetic** | Visual/copy issue only. | Vietnamese label overflows its card at 1.3× font scale. |

**Severity is what the defect does; priority is when we fix it.** They are set independently.

---

## 6. Test-case format

Every case in every plan uses this shape. Deviating makes runs harder to compare.

> ### `E2E-Mxx-nn` — Title of the case
> **Priority** P0 · **Type** Happy path · **Est.** 4 min
>
> **Preconditions** — what must be true before step 1.
> **Test data** — the exact values to use.
>
> | # | Action | Expected result |
> |---|---|---|
> | 1 | … | … |
>
> **Backend verification** — the API call or SQL that proves the state really changed.
> **Pass criteria** — the single sentence that decides pass/fail.

**Type** is one of: `Happy path`, `Negative`, `Edge`, `Recovery`, `Cross-client`, `Persistence`.

### Why "Backend verification" is mandatory

The mobile app caches aggressively (AsyncStorage for the widget, in-memory dashboard state, an
optimistic notification cache). **A screen showing the right number is not evidence the write
landed.** Every case that writes data must confirm it server-side — either by a `curl` through the
gateway with the tester's own token, or by killing and relaunching the app so all state is re-fetched.
Where a case can be verified by relaunch alone, it says so.

---

## 7. Entry and exit criteria

### Entry criteria — do not start a run until all are true
1. The build under test is installed on the target device and its version/commit is recorded.
2. All backend services are up: `GET /api/v1/auth/health` (or the equivalent smoke in
   [06-Development/03-Testing-and-Accounts §2](../06-Development/03-Testing-and-Accounts.md)) returns 200.
3. Fresh test accounts exist per [01-Test-Environment](01-Test-Environment-Builds-and-Data.md) — never
   the seeded `*.dev@mhsa.local` ones.
4. The device has network, notification permission granted, and (for M05) physical-activity permission.
5. The known-issues list in the [README](README.md#️-known-issues-every-tester-must-read-first) has been read.

### Exit criteria — the phase is done when
1. Every **P0** case has been executed and passed on the release-candidate build.
2. No open **S1** defects, and no **S2** defect on a P0 workflow.
3. Every executed case has a recorded result, device, build and date.
4. Any case marked *Blocked* has a written reason and an owner.
5. The traceability matrix ([02](02-Requirements-Traceability-Matrix.md)) shows no showcase feature
   with zero passing coverage.

---

## 8. Recording a run

Results go in the plan's own **Run log** table at the bottom of each file. One row per execution:

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|

Then update the status board in the [README](README.md#execution-status-board).

Screenshots and screen recordings for failures go in `09-Testing/evidence/<plan-id>/<case-id>-<date>.png`.
Create the folder on first use; it is intentionally not committed with placeholder files.

---

## 9. Defect report template

Paste this into the issue tracker (or a `defects/` note if none is in use):

```
ID:          DEF-<nnn>
Title:       <one line, the symptom, not the guess at the cause>
Found by:    E2E-Mxx-nn
Severity:    S1 | S2 | S3 | S4
Priority:    P0 | P1 | P2
Build:       <apk version / git commit of mobile + backend>
Device:      <model, Android version>
Account:     <test account email, profileId>
Timestamp:   <ISO-8601, with timezone — needed to find the server log line>

Steps to reproduce:
  1.
  2.

Expected:
Actual:
Frequency:   always | intermittent (n of m)
Evidence:    <screenshot / recording path, relevant logcat + server log excerpt>
Notes:       <anything ruled out; e.g. "not the known STOMP vhost bug — CONNECT succeeds">
```

**Always capture the timestamp with timezone.** The backend logs in UTC, the device shows ICT
(UTC+7); without a precise timestamp, correlating a mobile symptom to a server log line costs far
more than writing it down did.

---

## 10. Handling flaky results

A case that fails once and passes on retry is **not** a pass. Record it as *Intermittent* with the
observed rate (`2 of 5`). Two failure modes in this system are known to be genuinely rate-based:

- Anything touching a **first request after a long idle** — services lazily fetch JWKS on first token
  verification, and a cold container can take seconds. Retry once; if the retry succeeds instantly,
  note "cold start" rather than filing a defect.
- **Real-time chat** — the STOMP relay bug is fixed, but reconnection after a network flip is a real
  scenario with real timing. [M10](E2E-Mobile/E2E-M10-Social-Friends-and-Realtime-Chat.md) tests it
  explicitly rather than tolerating it silently.

Compare **rates across builds**, not single runs. A single red run is not evidence of a regression.

---

## 11. Tooling used during a run

| Purpose | Tool | Note |
|---|---|---|
| Device logs | `adb logcat -s ReactNativeJS:V AndroidRuntime:E` | The app logs STOMP connect/disconnect and API errors here. |
| Backend verification | `curl` through `https://umatter-apcs.duckdns.org` with the tester's own bearer token | Never use a hardcoded token; `axiosClient.ts` has a `HARDCODED_TEST_TOKEN` hook that must stay empty in any build under test. |
| Server logs | `docker compose logs -f <service>` on the VM | UTC timestamps. |
| Push testing | Firebase console → Cloud Messaging, or the real booking flow | Only `appointment.booked` is a live producer. |
| DB checks | `psql` into the owning service's database | Database-per-service — check the right one. |
| Network capture (optional) | Charles / mitmproxy with the device proxied | Release builds pin nothing, but HTTPS is required — cleartext is blocked at the manifest level. |

---

## 12. A note on test data hygiene

Test runs write real rows into the deployed databases. Two rules:

1. **Prefix every test artefact** so it can be found and cleaned: journal titles start with `QA-`,
   friend-request accounts use `+qa` addressing, appointment cancel reasons start with `QA test`.
2. **Never test destructive flows on an account used for the demo.** Keep the demo account separate
   from the QA accounts, and record which is which in
   [01-Test-Environment](01-Test-Environment-Builds-and-Data.md).
