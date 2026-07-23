# E2E-M11 — Daily Trophy & Streaks

> **Showcase §8** — *"**Cúp tracking hôm nay**: complete your daily categories — nutrition, sleep,
> diary, steps — to earn a bronze, silver, or gold cup. 'Tuyệt vời! Hoàn thành hôm nay 🎉'"*
> **Plan priority: P1.** Gamification is the motivational spine of the product: it is what turns
> tracking (the feature everything else depends on) into a habit. It is also entirely **client-side
> derived**, which makes it uniquely prone to disagreeing with the data it claims to summarise.

| | |
|---|---|
| **Scope** | The 4 tracking categories, bronze/silver/gold tiers, the celebration sheet, the trophy's agreement with actual server data, the day boundary, the journal streak / longest-streak showcase, profile statistics |
| **Out of scope** | The individual logging flows themselves (→ [M01](E2E-M01-Mood-Check-In-and-Focus-Mode.md), [M02](E2E-M02-Emotional-Journal.md), [M04](E2E-M04-Sleep-and-Nutrition-Tracking.md), [M05](E2E-M05-Automatic-Step-Counting.md)) |
| **Services** | Tracking (diaries, foods, sleeps, steps), Dashboard — **all trophy logic is on the client** |
| **Screens** | `ProfileScreen` → `DailyTrackingTrophy`, `TrophyShowcase`, `TrackingCelebrationSheet` |
| **Est. duration** | 40 min + a day boundary |
| **Cases** | 8 |

---

## Preconditions

1. TEEN-A logged in, on a device where you can log all four categories today (steps needs the sensor —
   see [M05](E2E-M05-Automatic-Step-Counting.md)).
2. **Start this plan on a day with no tracking logged yet.** A partially logged day makes the tier
   transitions impossible to observe.
3. The other tracking plans (M01/M02/M04/M05) have already been rehearsed, so you can log each
   category quickly.

## The rules as implemented

| Categories logged today | Tier |
|---|---|
| 0–1 | none (prompt: *"Log at least 2 categories today to earn the bronze cup!"*) |
| 2 | 🥉 **bronze** |
| 3 | 🥈 **silver** |
| 4 | 🥇 **gold** |

The four categories are **nutrition, sleep, diary, steps** — note that the **home-screen mood
check-in is *not* one of them**, even though it is the app's headline habit. Verify this and record
it (`M11-06`); it is a plausible product oversight worth raising, not a code bug.

**How "today" is decided:** each record resolves to a calendar day from its explicit `entryDate`
(first 10 characters) where present, falling back to a timestamp (`createdAt`, `wakeTime`,
`loggedAt`). Comparison is against the **device's local** date.

**Two layers of state, and this matters:**
1. `getTodayTrackingStatus(...)` derives the truth from **server data** fetched on Profile focus.
2. A module-level in-memory cache (`todayTrackingCache`) tracks what you have logged during this app
   session, so the celebration sheet can fire instantly. Profile **seeds** that cache from the server
   status on focus.

The cache is **in memory only** — it does not survive an app restart, and it is reset on a new
calendar day. `M11-05` tests the seam between the two layers, which is exactly where a wrong tier
would come from.

---

## Test cases

### `E2E-M11-01` — Earn bronze, silver and gold in sequence
**Priority** P1 · **Type** Happy path · **Est.** 12 min

**Preconditions** — Nothing logged today.

| # | Action | Expected result |
|---|---|---|
| 1 | Open Profile and view the daily trophy | No cup; the prompt says to log at least 2 categories today |
| 2 | Check the category row | All four badges (nutrition, sleep, diary, steps) show as **pending** |
| 3 | Log **nutrition** ([M04-04](E2E-M04-Sleep-and-Nutrition-Tracking.md)) | A celebration sheet appears showing 1 of 4; still no cup |
| 4 | Log **sleep** | Celebration shows 2 of 4 and a **🥉 bronze** cup is earned |
| 5 | Return to Profile | The trophy shows bronze; nutrition and sleep badges are **done**, the other two pending |
| 6 | Log a **diary** entry | **🥈 silver** |
| 7 | Ensure **steps** are logged for today (walk, then open the steps screen to sync) | **🥇 gold**, with the completion message (*"Tuyệt vời! Hoàn thành hôm nay 🎉"*) |
| 8 | Return to Profile | Gold cup; all four badges done |

**Pass criteria** — The tier advances at exactly 2, 3 and 4 categories, and the per-category badges
match what was actually logged.

---

### `E2E-M11-02` — The trophy agrees with the server 🔑
**Priority** P1 · **Type** Persistence · **Est.** 6 min

The trophy is computed on the client from fetched data — so the real question is whether it agrees
with the database.

| # | Action | Expected result |
|---|---|---|
| 1 | Note the current tier and which badges are marked done | Recorded |
| 2 | Query all four collections for today | See the commands below |
| 3 | Compare | Exactly the categories with a record dated today are marked done — no more, no fewer |
| 4 | **Force-stop the app** and relaunch, open Profile | Same tier as before the restart (the in-memory cache is gone, so this reads purely from the server) |
| 5 | Log in as TEEN-A on a **different** device | The same tier is shown, since it derives from the same server data |

```bash
D=$(date +%F)
for R in diaries foods sleeps steps; do
  echo "== $R"; curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/$R/{profileId}" \
    -H "Authorization: Bearer $TOKEN" | jq --arg d "$D" '[.[] | select((.entryDate // "")[0:10]==$d)] | length'
done
```

**Pass criteria** — The tier and badges match the server data exactly, before and after a restart, on
any device.

**Note** — Step 4 is the important one. A trophy that is right only until you restart means the
celebration is driven by session state rather than data, and the user's "progress" is an illusion.
**S2**.

---

### `E2E-M11-03` — The celebration sheet
**Priority** P2 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Log a category that takes you from 1 → 2 | The celebration sheet appears with the bronze milestone |
| 2 | Read it | It shows the count out of 4 and which categories remain |
| 3 | Dismiss it | Closes cleanly; you return to the screen you were on |
| 4 | Re-save the **same** category again | Record whether the sheet re-fires — the cache is idempotent per category, so it should not celebrate the same category twice |
| 5 | Reach 4 of 4 | The sheet shows the completion message rather than a "keep going" message |

**Pass criteria** — The sheet fires on genuine progress, shows the right count, and does not
re-celebrate an already-counted category.

---

### `E2E-M11-04` — Day boundary: the trophy resets ⏰ *overnight case*
**Priority** P1 · **Type** Edge · **Est.** 5 min active, overnight wait

| # | Action | Expected result |
|---|---|---|
| 1 | End a day with a gold cup | Recorded |
| 2 | The next morning, open Profile | The daily trophy has **reset** — the new day starts at 0 categories |
| 3 | Check the category badges | All pending again |
| 4 | Log one category | Count is 1, not 5 — yesterday's does not carry over |
| 5 | Check yesterday's data is intact | The server still holds yesterday's four records |

**Pass criteria** — The daily trophy resets at the local day boundary without destroying history.

**Note** — Do not fake this with the device clock. Both the trophy's date key and the records'
`entryDate` derive from the device's local date, so a clock jump can produce a misleading pass. If a
real overnight is impossible, mark it **Blocked**.

---

### `E2E-M11-05` — Session cache vs. server truth
**Priority** P1 · **Type** Edge · **Est.** 6 min

This case targets the seam between the in-memory cache and the fetched status.

| # | Action | Expected result |
|---|---|---|
| 1 | Log two categories on **device 1** (bronze) | Bronze shown |
| 2 | Without restarting device 1, log a third category **from a second device / via API** | — |
| 3 | On device 1, navigate away from Profile and back (triggering a focus refetch) | The tier updates to **silver** — the cache is re-seeded from the server, not stuck at the session's own count |
| 4 | Delete one of today's records via API, then refocus Profile | The tier **decreases** accordingly — the trophy reflects the current truth, not a high-water mark |
| 5 | Force-stop and relaunch | Same tier |

**Pass criteria** — The client's cache never outranks the server's truth after a refocus, in either
direction.

---

### `E2E-M11-06` — Confirm which categories count
**Priority** P2 · **Type** Edge · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | On a fresh day, log **only** a home-screen mood check-in | Record whether the trophy count moves |
| 2 | Log **only** a breathing session | Record whether the count moves |
| 3 | Compare against the four documented categories | Only nutrition, sleep, diary and steps should count |
| 4 | Record the finding | — |

**Pass criteria** — The implemented categories match the documented four. This case exists to make the
mood/breathing exclusion an explicit, recorded decision.

**Note** — Excluding the **mood check-in** is worth flagging to the team. Showcase §1 calls the mood
check-in *"the heart of uMatter"* and §11 calls daily reflection *"the core"*, yet a user can log
their mood every 90 minutes all day and earn no cup for it. That is a **product** inconsistency
(S3-equivalent), not a code defect — but it is exactly the kind of thing a council member notices.

---

### `E2E-M11-07` — Streak showcase and longest streak
**Priority** P1 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open Profile and find the streak/trophy showcase | It displays a longest-streak figure |
| 2 | Cross-check against the journal streak on the journal overview ([M02-07](E2E-M02-Emotional-Journal.md)) | The two numbers are consistent with the same underlying entries |
| 3 | With entries on 3 consecutive days, verify the figure | Matches the actual run |
| 4 | Break the streak (a day with no entry), then journal again | The **current** streak restarts while the **longest** figure is preserved |
| 5 | Cross-check against the raw entry dates from the API | The longest run in the data matches what is displayed |

**Pass criteria** — Current and longest streaks are both correct against the raw data, and the longest
is not lost when the current one breaks.

**Note** — There is **no backend streak service**; everything here is derived on the client. That also
means the dormant `streak.milestone` notification flow does **not** exist — no push will ever fire for
a streak. Do not test for one.

---

### `E2E-M11-08` — Profile statistics
**Priority** P2 · **Type** Happy path · **Est.** 4 min

> Showcase §8: *"Your Profile shows off your achievements: journal streak, AI chat sessions, and
> 'member since'."*

| # | Action | Expected result |
|---|---|---|
| 1 | Open Profile | Journal streak, AI chat session count and "member since" are all displayed |
| 2 | Check the AI session count | Matches the number of sessions from `GET /api/v1/ai/chat/sessions` |
| 3 | Start a new AI conversation, return to Profile | The count increments (after a refocus) |
| 4 | Check "member since" | Matches the account's registration date, formatted in Vietnamese |
| 5 | Check a brand-new account | Zeros and today's date — no `NaN`, no "Invalid Date", no blank |

**Pass criteria** — Every profile statistic matches its underlying data, including on a fresh account.

---

## Exit criteria for M11

- `M11-01` passes — all three tiers are reachable.
- **`M11-02` passes** — the trophy agrees with the server, including after a restart.
- `M11-04` executed on a real day boundary, or Blocked with a reason.
- `M11-06`'s finding recorded and reviewed.
- No **S1** defect open (a wrong trophy is rarely S1, but a trophy that hides real logged data is).

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| All trophy/streak logic is client-side; no backend achievement service | ✅ By design |
| No push notification for streaks | ⚠️ Known — the `streak.milestone` consumer was deleted; nothing publishes it |
| The trophy cache is in-memory and lost on restart | ✅ By design — the server-derived status is the source of truth on focus |
| Mood check-ins and breathing do not count toward the daily cup | ⚠️ Confirm in `M11-06` and raise as a product question |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
