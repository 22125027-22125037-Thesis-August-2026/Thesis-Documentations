# E2E-M13 — Privacy: Data-Access Grants & Consent

> **Showcase §10** — *"**You decide who sees your data.** Your journal, mood, and sleep are **yours**.
> A therapist can only view them **if you explicitly grant permission** — and you can revoke it at any
> time (*'Người được xem nhật ký của bạn'*)."*
> **Showcase §11** — *"Your therapist, once you grant access, can see your mood flow across the week…
> A trusted friend, if you allow it, can gently look out for you."*
>
> **Plan priority: P0.** This is a **privacy** claim about the health data of minors. A false negative
> here (data visible without consent) is the most serious defect the project can ship. It is also the
> claim most likely to be probed hard by a thesis council.

| | |
|---|---|
| **Scope** | Granting access to a friend, per-category scope enforcement, what a grantee actually sees, granting a therapist, revocation, grant expiry, the AI companion as a grantee, and enforcement at the API layer |
| **Out of scope** | The AI's use of grounded data (→ [M07](E2E-M07-AI-Companion-Ban-Tam-Giao.md), which tests the *behaviour*; this plan tests the *permission*) |
| **Services** | Auth (grant of record), Tracking (**enforcement** + replica), Social, Therapist web UI |
| **Screens** | `FriendProfileScreen`, `DailyLogsSection`, social `ChatScreen` header, AI `ChatScreen` toggle |
| **Est. duration** | 60 min, two clients |
| **Cases** | 10 |

---

## Preconditions

1. **TEEN-A** (the data owner) with real tracking history: sleep, food and journal entries.
2. **TEEN-B** on a second device, already friends with TEEN-A ([M10-03](E2E-M10-Social-Friends-and-Realtime-Chat.md)).
3. **THERAPIST-T** with a completed or booked appointment with TEEN-A, signed in to the web UI.
4. Both `profileId`s and both bearer tokens to hand — several cases verify at the API layer, which is
   the only way to prove enforcement rather than UI hiding.

## The grant model as implemented

| Property | Value |
|---|---|
| Grant of record | Auth service — `POST /api/v1/auth/grants` |
| **Enforcement** | **Tracking service**, using a replicated copy of the grant |
| Scope format | CSV of category tokens, or the shorthand `READ_ALL` |
| Category tokens | `READ_MOOD`, `READ_SLEEP`, `READ_FOOD`, `READ_JOURNAL`, `READ_STEPS`, `READ_BREATHING` |
| Offered in the friend UI | **Only 3**: `READ_SLEEP` (Giấc ngủ), `READ_FOOD` (Ăn uống), `READ_JOURNAL` (Nhật ký) |
| Friend/therapist grant expiry | **30 days** from granting |
| AI companion grant | `READ_ALL`, **no expiry**, granted from the chat toggle |
| Default | **Deny.** No grant → 403 from Tracking |
| Revocation | `DELETE /api/v1/auth/grants/{granteeProfileId}` |

### ⚠️ Two facts that decide how you test this

1. **Enforcement is server-side, in Tracking.** So the real test is not "does the app hide the card"
   — it is "does the API return 403". Every case below verifies at the API layer. A UI that hides
   data while the API serves it is **S1**.
2. **Seeded grants do not work.** The 43 grants in the seed exist only in `auth_db`; they bypass the
   event pipeline, so Tracking's replica never receives them and they grant nothing until the nightly
   reconcile (23:30 ICT). **Always create grants through the app** — and check this before a demo.

---

## Test cases

### `E2E-M13-01` — Default deny: a friend sees nothing 🔑
**Priority** P0 · **Type** Negative · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Confirm no grant exists A→B | `GET /api/v1/auth/grants/status/{B_profileId}` as A → `iGaveThemAccess: false` |
| 2 | On **B's** device, open A's friend profile | Identity only — **no** sleep, food or journal data |
| 3 | Attempt to read A's data directly as B, at the API | Each of these must return **403**: |

```bash
for R in sleeps foods diaries moods steps breathing; do
  printf "%-10s " "$R"
  curl -s -o /dev/null -w "%{http_code}\n" \
    "https://umatter-apcs.duckdns.org/api/v1/tracking/$R/{A_profileId}" \
    -H "Authorization: Bearer $TOKEN_B"
done
```

| # | Action | Expected result |
|---|---|---|
| 4 | Review the six status codes | **All 403.** Any 200 is an immediate **S1** stop-the-line defect |
| 5 | Repeat as an unrelated third account | Also all 403 |

**Pass criteria** — With no grant, no other account can read any of the owner's tracking data through
the API, not merely through the UI.

---

### `E2E-M13-02` — Grant all three categories to a friend
**Priority** P0 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | On **A's** device, open the chat with B, then open B's profile from the header | The profile shows sharing controls |
| 2 | Observe the category list | Three options: *Giấc ngủ*, *Ăn uống*, *Nhật ký* — all selected by default |
| 3 | Confirm the grant | A confirmation step appears before anything is shared |
| 4 | Confirm | Success; the screen updates to show access is granted |
| 5 | Verify server-side | `GET /api/v1/auth/grants/status/{B_profileId}` as A → `iGaveThemAccess: true`, `accessScope` = `READ_SLEEP,READ_FOOD,READ_JOURNAL`, `expiresAt` ≈ **30 days** out |
| 6 | On **B's** device, open A's profile | The daily-logs section now renders sleep, food and journal cards with A's **real** data |
| 7 | Cross-check one value against A's own screen | Identical |

**Pass criteria** — A grant created in the app is visible server-side with the right scope and expiry,
and the grantee immediately sees exactly the granted data.

---

### `E2E-M13-03` — Per-category scope is enforced 🔑
**Priority** P0 · **Type** Edge · **Est.** 8 min

This is the case that proves the grant is a real permission rather than an on/off switch.

| # | Action | Expected result |
|---|---|---|
| 1 | Revoke any existing grant A→B | Status back to `false` |
| 2 | Grant **only** *Giấc ngủ* (`READ_SLEEP`) | Scope on the server is exactly `READ_SLEEP` |
| 3 | As B, call the six endpoints again | `sleeps` → **200**; `foods`, `diaries`, `moods`, `steps`, `breathing` → **403** |
| 4 | On B's device, view A's profile | **Only** the sleep card renders — no food, no journal |
| 5 | Widen the grant to `READ_SLEEP,READ_JOURNAL` | Re-grant with both selected |
| 6 | Re-run the six calls | `sleeps` **200**, `diaries` **200**, `foods` still **403** |
| 7 | On B's device, refresh A's profile | Sleep and journal cards render; food does not |
| 8 | Narrow back to `READ_FOOD` only | `foods` 200; `sleeps` and `diaries` back to **403** |

**Pass criteria** — Every category responds 200 **if and only if** its token is in the current scope,
in all three directions (grant, widen, narrow).

**Note** — Step 8's *narrowing* is the one most likely to fail. A scope change that only ever adds
permission, never removes it, is **S1** — it means a user who thinks they have reduced sharing has not.

---

### `E2E-M13-04` — What the grantee actually sees
**Priority** P1 · **Type** Happy path · **Est.** 6 min

> Showcase §11's example: *"a trusted friend… noticing from your shared food diary that you've been
> skipping meals, and checking in."*

| # | Action | Expected result |
|---|---|---|
| 1 | With a full grant in place, open A's profile as B | Cards render for each granted category |
| 2 | Compare each rendered value against A's own screens | Values match exactly — no rounding drift, no stale data |
| 3 | Have A add a **new** entry, then refresh on B | The new entry appears — the grantee sees live data, not a snapshot |
| 4 | Check the journal card specifically | Record **how much** of the journal is exposed: title only, or full body and photos? This is the most sensitive content in the app and the answer belongs in the thesis |
| 5 | Check for anything **not** granted appearing anywhere | Nothing outside the scope is visible on any screen |

**Pass criteria** — The grantee sees accurate, live data for granted categories and nothing else.
Step 4's finding is recorded.

---

### `E2E-M13-05` — Grant a therapist and see the data on the web UI
**Priority** P0 · **Type** Cross-client · **Est.** 8 min

| # | Action | Expected result |
|---|---|---|
| 1 | On A's device, open the **therapist consultation** chat channel | The channel header opens a profile view for the therapist |
| 2 | Grant access from there, selecting all three categories | Confirmation, then granted |
| 3 | Verify server-side | Grant exists A→THERAPIST-T with the expected scope |
| 4 | On the **therapist web UI**, open A as a patient | Sleep, food and journal are now visible |
| 5 | Compare the web UI's values against A's phone | Identical |
| 6 | Check a category **not** granted (e.g. mood) | Not visible on the web UI, and the corresponding API call is refused |

**Pass criteria** — A grant created on the phone makes the data visible in the therapist's web UI, and
only within scope.

**⚠️ Discoverability finding to record.** Granting a therapist is only reachable **through the
consultation chat channel's header** — there is no dedicated "share my data with my therapist" action
in the booking or appointment flow. Record the exact tap path you had to use. If a tester who knows
the app struggles to find it, a 15-year-old will not find it at all, and the showcase's *"once you
grant access"* premise quietly never happens. Raise as **S3** usability with a strong argument for
**S2**.

---

### `E2E-M13-06` — Revocation takes effect immediately 🔑
**Priority** P0 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | With a full grant A→B active and B currently **viewing** A's data | Data visible |
| 2 | On A's device, revoke access | Confirmation, then revoked |
| 3 | Verify server-side | `iGaveThemAccess: false` |
| 4 | On B's device, refresh A's profile | The data cards are **gone** |
| 5 | As B, call the six API endpoints | All **403** again |
| 6 | Time the gap between step 2 and step 5 | Enforcement is immediate — **not** dependent on the nightly reconcile |
| 7 | Repeat for the therapist grant, refreshing the web UI | Data disappears there too |

**Pass criteria** — Revocation is immediate and complete at the API layer for every category and every
grantee type.

**Note** — Step 6 is the crux. Because enforcement uses a **replica** of the grant in Tracking, a
revocation that only lands in `auth_db` would leave the data readable until the nightly reconcile.
Measure the delay and record it. Anything beyond a few seconds is **S1** for a privacy control the app
describes as "revoke at any time".

---

### `E2E-M13-07` — Grant expiry
**Priority** P2 · **Type** Edge · **Est.** 5 min

Friend/therapist grants carry a **30-day** expiry; the AI grant carries **none**.

| # | Action | Expected result |
|---|---|---|
| 1 | Create a friend grant and read `expiresAt` | ≈ 30 days from now |
| 2 | Confirm the AI grant's expiry | `null` — persists until turned off |
| 3 | Check whether the expiry is **shown to the user** anywhere | Record it — a user who does not know their sharing lapses in 30 days will be surprised in either direction |
| 4 | Simulate expiry (backdate `expiresAt` in the DB, or create a grant with a short expiry via API) | Once expired, the six endpoints return **403** again |
| 5 | Check the grantee's UI after expiry | Data disappears; no stale cached view |

**Pass criteria** — Expired grants stop granting, and the expiry behaviour is documented.

**Note** — Step 4 requires either DB access or a direct API call; record which you used, since a DB
edit bypasses the event pipeline and could itself explain a wrong result.

---

### `E2E-M13-08` — The AI companion is subject to the same model
**Priority** P0 · **Type** Cross-feature · **Est.** 5 min

Cross-references [M07-06/07](E2E-M07-AI-Companion-Ban-Tam-Giao.md); here we check it from the
**permission** side.

| # | Action | Expected result |
|---|---|---|
| 1 | With the AI toggle **off**, check the grant status for `a1000000-a1a1-4a1a-8a1a-a1a1a1a1a1a1` | `iGaveThemAccess: false` |
| 2 | Turn the toggle on | A grant appears with scope `READ_ALL` and **no** expiry |
| 3 | Confirm the AI is an ordinary grantee | It appears in the same grant model as a human grantee — one mechanism, not a special case |
| 4 | Turn the toggle off | The grant is deleted, not merely marked inactive in the app |
| 5 | Confirm the toggle survives a restart | Reads the server's state on mount |

**Pass criteria** — The AI's access is a normal grant, created and destroyed by the toggle, and
inspectable through the same API as any other.

---

### `E2E-M13-09` — Consent cannot be bypassed
**Priority** P0 · **Type** Negative · **Est.** 8 min

The adversarial pass. Everything here should fail closed.

| # | Action | Expected result |
|---|---|---|
| 1 | As B (no grant), call A's tracking endpoints directly with a valid token | **403** on every category |
| 2 | As B, call A's endpoints with **no** Authorization header | **401** |
| 3 | As B, call with a **tampered** token (alter one character of the signature) | **401** — not 200, not 500 |
| 4 | As B with a `READ_SLEEP`-only grant, call `diaries` | **403**, not a filtered 200 |
| 5 | As **THERAPIST-T** with no grant to A, call A's endpoints | **403** — being someone's therapist is not itself consent |
| 6 | Call another patient's data as THERAPIST-T (a patient who never booked them) | **403** |
| 7 | As A, call **your own** data | **200** — the owner is never blocked by the grant system |

**Pass criteria** — Every unauthorised path returns 401/403, and the owner's own access is unaffected.

**Note** — Step 5 deserves emphasis in the thesis: *a professional relationship is not consent*. If a
therapist can read a patient's journal purely because an appointment exists, showcase §10 is false.
**S1.**

---

### `E2E-M13-10` — Is there a "who can see my data" view?
**Priority** P1 · **Type** Edge · **Est.** 5 min

Showcase §10 names a specific affordance: *"Người được xem nhật ký của bạn"* — the people who can see
your journal.

| # | Action | Expected result / record |
|---|---|---|
| 1 | Search the Profile tab for a privacy/data screen | Record whether one exists |
| 2 | Search for any consolidated list of active grants | Record |
| 3 | Grant access to **two** different friends | Both grants exist server-side (`GET /api/v1/auth/grants/{profileId}`) |
| 4 | Try to see both grants **in one place in the app** | Record whether this is possible |
| 5 | Try to revoke one **without** navigating to that specific friend's profile | Record whether this is possible |

**Pass criteria** — This case is a **finding-recorder**; it passes when the answer is documented.

**⚠️ Expected finding.** In the build inspected, grants can only be created or revoked **per person,
from that person's profile screen**. The API to list grants (`listMyGrants`, `listGrantsReceived`)
exists but no screen calls it, and the Profile tab's privacy entry has no destination. That means:

- there is **no** single place showing everyone who can see your data;
- to revoke, you must remember who you granted and navigate to each one;
- if you unfriend someone, or simply forget, the grant may persist unseen for its full 30 days.

That is a genuine shortfall against the §10 promise, and it is a small, well-scoped piece of work
(one screen over an API that already exists). Rate **S2** and recommend it as the highest-value
privacy fix before the defence.

---

## Exit criteria for M13

**Like [M12](E2E-M12-Crisis-Safety-and-Emergency-Support.md), this plan has strict exits.**

- **`M13-01`, `M13-03`, `M13-06` and `M13-09` all pass.** These four are the privacy claim. If any
  fails, the app should not be demonstrated as privacy-respecting until it is fixed.
- `M13-05` passes and its discoverability finding is recorded.
- `M13-10`'s finding is recorded and put to the team as a decision.
- **Zero** open S1 defects. A single unauthorised 200 is a stop-the-line event.
- Before any demo: create a **fresh** grant through the app and verify it works — seeded grants do not.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| Seeded grants grant nothing until the nightly reconcile | ⚠️ **Known** — always grant through the app |
| Only 3 of 6 category tokens are offered in the friend UI | ⚠️ Record — mood, steps and breathing cannot be shared with a friend from the app |
| No consolidated "who can see my data" screen | ⚠️ Confirm in `M13-10` and escalate |
| Granting a therapist is only reachable via the consult chat header | ⚠️ Confirm in `M13-05` and escalate |
| Enforcement lives in Tracking, not Auth | ✅ By design — verify at the API, not the UI |
| AI grants are `READ_ALL` with no expiry; human grants are scoped with 30-day expiry | ✅ By design |

## Run log

| Date | Build / commit | Devices + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
