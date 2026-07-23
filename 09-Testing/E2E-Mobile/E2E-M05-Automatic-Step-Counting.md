# E2E-M05 — Automatic Step Counting

> **Showcase §4** — *"👟 **Bước chân** (Steps) — **Automatically counts your steps** in the background,
> with a daily goal, distance & calories, and a "Đã đạt mục tiêu! 🎉" celebration when you hit it."*
> **Plan priority: P1.** This is the only feature that produces data **without the user doing
> anything**, which makes it the only one that can be silently wrong for weeks. Showcase §11 also
> makes it a clinical claim: the counsellor consulted for the project wanted step data so a therapist
> can *"track your activity and assign suitable exercises"* — so accuracy is not cosmetic.

| | |
|---|---|
| **Scope** | `ACTIVITY_RECOGNITION` permission, hardware sensor availability, live step updates, the daily-baseline algorithm, midnight rollover, reboot handling, backend upsert, goal/distance/calorie display, 7-day chart |
| **Out of scope** | Other tracking screens (→ [M04](E2E-M04-Sleep-and-Nutrition-Tracking.md)), trophy credit (→ [M11](E2E-M11-Daily-Trophy-and-Streaks.md)) |
| **Services** | Tracking (`/api/v1/tracking/steps`), Dashboard, device step-counter sensor |
| **Screens** | `StepMainScreen`, home mini-dashboard |
| **Est. duration** | 45 min active + real walking + one overnight |
| **Cases** | 8 |

---

## Preconditions

1. **A physical Android device with a hardware step-counter sensor.** An emulator cannot run this
   plan; `stepTracker.isAvailable()` will return false and every case is Blocked.
2. Android 10 (API 29) or newer to exercise the runtime permission path. On older Android the
   permission is granted at install time and `M05-01` degenerates.
3. TEEN-A logged in.
4. A pedometer or a second phone to walk alongside for `M05-03` cross-checking.
5. Device timezone ICT; clock automatic (the baseline keys off the **local** calendar date).

## Reference values baked into the build

| Constant | Value | Where it shows |
|---|---|---|
| Daily goal | **6 000** steps | Progress ring, "remaining" text, goal-reached state |
| Distance per step | **0.0008 km** (0.8 m) | `distanceKm = steps × 0.0008`, shown to 2 dp |
| Calories per step | **0.04 kcal** | `calories = steps × 0.04` |
| Source tag | `DEVICE_SENSOR` | Sent on every upsert |

## How the counting actually works — read this before interpreting results

The Android sensor reports a **cumulative count since boot**, not a daily figure. The app derives
today's steps as `cumulative − baseline`, where the baseline is stored in AsyncStorage under
`@steps/daily_baseline` as `{ date, baseline }` and is re-anchored in exactly two situations:

1. **New local day** — the stored date ≠ today → baseline = current cumulative.
2. **Cumulative went backwards** (device rebooted, sensor reset) → baseline = current cumulative.

Sync to the backend is an **upsert keyed on `entryDate`**, called on app foreground and on the steps
screen. Two consequences worth holding in mind while testing:

- Steps taken **while the app has never been opened that day** are still counted, because the sensor
  keeps counting and the first foreground of the day computes the delta — but a reboot before that
  first foreground loses the pre-reboot portion (see `M05-05`).
- There is **no background service**. "Automatically counts in the background" means *the sensor
  counts*; the app reconciles when it next runs.

---

## Test cases

### `E2E-M05-01` — Permission request and denial handling
**Priority** P1 · **Type** Happy path + Negative · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Fresh install; open the steps screen for the first time | The Android `ACTIVITY_RECOGNITION` ("Physical activity") permission prompt appears |
| 2 | **Deny** it | The screen shows an explanatory state — not a crash, not a silent `0`, not an infinite spinner |
| 3 | Look for a route to fix it | The user can understand they need to grant the permission (in-app text pointing at settings is enough) |
| 4 | Grant it via Android settings and return to the app | The screen begins counting without needing a reinstall |
| 5 | Reopen the screen later | No repeated prompt once granted |

**Pass criteria** — Denial degrades gracefully and is recoverable without reinstalling.

**Note** — A denied permission that renders as a confident `0 bước` is **S2**: it is indistinguishable
from "you did not move today", which is a misleading health statement.

---

### `E2E-M05-02` — Live counting while the screen is open
**Priority** P1 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open the steps screen and note the displayed count | Value recorded as `S₀` |
| 2 | Walk **100 steps**, counting deliberately, with the screen on and the app in the foreground | The displayed count increases live, without needing a manual refresh |
| 3 | Compare to `S₀ + 100` | Within a reasonable sensor tolerance (±10 %). Hardware step detection is inherently approximate — a systematic 2× or 0.5× error is a defect, ±8 steps is not |
| 4 | Observe distance and calories | `distance ≈ steps × 0.0008 km` (2 dp) and `calories ≈ steps × 0.04 kcal` — recompute by hand from the displayed step count and confirm |
| 5 | Observe the progress ring | Proportion matches `steps / 6000` |

**Pass criteria** — Live updates track real walking, and the derived distance/calorie figures are
arithmetically consistent with the shown step count.

---

### `E2E-M05-03` — Steps taken with the app closed are picked up
**Priority** P1 · **Type** Happy path · **Est.** 8 min

This is the actual "automatic" claim.

| # | Action | Expected result |
|---|---|---|
| 1 | Open the app, note the count `S₀`, then **background** it (home button — do **not** force-stop) | — |
| 2 | Walk ~200 steps with the phone in a pocket | — |
| 3 | Foreground the app and open the steps screen | The count has increased by roughly 200 — the reconciliation happens on foreground |
| 4 | Repeat, but this time **force-stop** the app before walking | On relaunch the count **still** includes the walk, because the sensor is cumulative and the baseline is persisted |
| 5 | Verify the backend after each foreground | Today's step record reflects the new total |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/steps/{profileId}" \
  -H "Authorization: Bearer $TOKEN" | jq '.[] | select(.entryDate=="'"$(date +%F)"'")'
```
`stepCount` matches the screen; `source` is `DEVICE_SENSOR`; `goal` is `6000`.

**Pass criteria** — Walking without the app in the foreground is counted, and the value reaches the
backend on the next foreground.

---

### `E2E-M05-04` — Midnight rollover resets to zero ⏰ *overnight case*
**Priority** P1 · **Type** Edge · **Est.** 5 min active, overnight wait

| # | Action | Expected result |
|---|---|---|
| 1 | Late in the evening, note today's count (`S_end`) and confirm it is on the backend | Recorded |
| 2 | Leave the phone overnight without force-stopping | — |
| 3 | The next morning, open the app | Today's count starts near **0** — not carrying `S_end` forward, and not negative |
| 4 | Walk 50 steps | The new day's count rises from ~0 |
| 5 | Query the backend | **Two separate records** exist, one per `entryDate`; yesterday's still holds `S_end` and was not overwritten |
| 6 | Check the 7-day chart | Yesterday and today are separate columns with the right values |

**Pass criteria** — The day boundary creates a new record and a fresh count, leaving yesterday's total
intact.

**Note** — Do **not** fake this by changing the device clock. The rollover interacts with the stored
baseline date *and* the backend's `entryDate`; a manual clock jump can produce a pass that a real
midnight would fail (or vice-versa). If a real overnight is impossible in the schedule, mark this case
**Blocked** and say so — a fabricated pass is worse than a gap.

---

### `E2E-M05-05` — Device reboot re-anchors the baseline
**Priority** P2 · **Type** Recovery · **Est.** 8 min

The sensor's cumulative counter resets to 0 on reboot. The app detects `cumulative < baseline` and
re-anchors.

| # | Action | Expected result |
|---|---|---|
| 1 | Note today's in-app count `S₀` (make sure `S₀ > 0`) | Recorded |
| 2 | Reboot the device | — |
| 3 | Open the app and the steps screen | The count does **not** go negative and does **not** show a wild value |
| 4 | Record what it shows | Expect it to restart near 0 for the remainder of the day — the pre-reboot steps are lost, because the baseline re-anchors to the post-reboot cumulative |
| 5 | Walk 50 steps | Counting resumes correctly from the new anchor |
| 6 | Check the backend | Today's record was **upserted downward** to the new value — confirm and record this |

**Pass criteria** — No negative or absurd count after a reboot, and counting resumes.

**Known limitation to record (not a defect to file)** — Because the upsert overwrites `entryDate`'s
record, a reboot mid-day can **reduce** the stored total for that day. It is a correct consequence of
the chosen algorithm and it is bounded (one day, one reboot), but it should be stated honestly rather
than discovered by a council member asking "what happens if I restart my phone?".

---

### `E2E-M05-06` — Goal reached celebration
**Priority** P2 · **Type** Happy path · **Est.** 5 min

Goal is **6 000** steps.

| # | Action | Expected result |
|---|---|---|
| 1 | Below the goal, read the progress text | Shows how many steps remain (`goal − steps`) |
| 2 | Cross 6 000 steps (walk, or use an account/day already above it) | The screen switches to the goal-reached state — *"Đã đạt mục tiêu! 🎉"* |
| 3 | Continue past the goal | The ring stays at 100 % (it clamps) while the raw step count keeps rising — no ring overflow |
| 4 | Check the remaining-steps figure at/above goal | Reads `0`, never a negative number |

**Pass criteria** — The celebration appears exactly at/after the goal, and no derived figure goes
negative past it.

**Practical tip** — 6 000 steps is roughly 45–60 minutes of walking. If you cannot walk it during the
test window, seed a day at/above goal through the API and verify the *display* logic on that day,
then note that the live-crossing transition was verified separately or not at all.

---

### `E2E-M05-07` — Sensor unavailable / unsupported device
**Priority** P2 · **Type** Negative · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Run the app on a device (or emulator) with **no** step-counter sensor | The steps screen shows an unavailable/empty state with an explanation |
| 2 | Confirm nothing is written | **No** step record is upserted for that day with a bogus `0` — check the backend |
| 3 | Confirm the rest of the app is unaffected | Home dashboard renders; other tracking works; the trophy simply cannot earn the steps category |

**Pass criteria** — An unsupported device degrades to an explained empty state and does not pollute
the backend with zeroes.

**Note** — A `0`-step record written on an unsupported device is worse than no record: it feeds the
7-day chart, the trophy, and (via a grant) a therapist's view with a false statement about the
patient's activity. Treat that as **S2**.

---

### `E2E-M05-08` — Backend sync failure is non-destructive
**Priority** P2 · **Type** Recovery · **Est.** 5 min

The sync helper swallows errors by design (it logs and returns) so a network blip never breaks the
screen.

| # | Action | Expected result |
|---|---|---|
| 1 | Enable aeroplane mode, open the steps screen, walk 50 steps | The **local** count still rises — sensor reading does not depend on the network |
| 2 | Observe the UI | No crash and no error alert storm; at most a quiet failure |
| 3 | Restore network and foreground the app | The next sync upserts the current total |
| 4 | Verify the backend | Today's record now matches the on-screen count — no lost day, no duplicate record |

**Pass criteria** — Offline counting continues locally and reconciles to the correct total once
connectivity returns.

---

## Exit criteria for M05

- `M05-01`, `M05-02`, `M05-03` pass — permission, live counting and background pickup work.
- `M05-04` is executed on a **real** overnight, or explicitly marked Blocked with a reason.
- `M05-07` passes — no false zeroes on unsupported hardware.
- Reboot behaviour (`M05-05`) is recorded with its limitation, whatever the outcome.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| No background service; totals reconcile on app foreground | ✅ By design — the *sensor* counts continuously, the app catches up |
| Pre-reboot steps are lost after a reboot | ✅ Consequence of the baseline algorithm — record it, don't file it |
| Distance/calories are linear constants (0.8 m, 0.04 kcal per step), not personalised | ✅ By design — no height/weight input exists |
| Goal is fixed at 6 000 and not user-editable | ✅ By design in this build |
| iOS returns 0 / unavailable | ✅ By design — Android-only implementation |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
