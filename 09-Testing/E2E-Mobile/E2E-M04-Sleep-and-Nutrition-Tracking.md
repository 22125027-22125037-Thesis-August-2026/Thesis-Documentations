# E2E-M04 — Sleep & Nutrition Tracking

> **Showcase §4** — *"uMatter knows mental health isn't just mood — it's sleep, movement, food, and
> breath."*
> **Plan priority: P1.** These two logs are what make the AI's replies personal and what a therapist
> reads first after mood. They are also the two features where the **showcase text and the shipped
> screens diverge**, so this plan tests what the app does *and* records the gap.

| | |
|---|---|
| **Scope** | Sleep log (date, bedtime, wake time, 5 quality levels), nutrition log (water in litres, meal count, description), one-log-per-day update semantics, 7-day trend charts, home mini-dashboards |
| **Out of scope** | Steps (→ [M05](E2E-M05-Automatic-Step-Counting.md)), breathing (→ [M06](E2E-M06-Breathing-and-Meditation.md)), trophy credit (→ [M11](E2E-M11-Daily-Trophy-and-Streaks.md)), therapist visibility (→ [M13](E2E-M13-Privacy-Data-Access-Grants.md)) |
| **Services** | Auth, Tracking (`/api/v1/tracking/sleeps`, `/api/v1/tracking/foods`), Dashboard |
| **Screens** | `SleepMainScreen`, `FoodMainScreen`, `MiniDashboardsSection` on Home |
| **Est. duration** | 45 min |
| **Cases** | 9 |

---

## Preconditions

1. TEEN-A logged in; `profileId` recorded.
2. Device timezone ICT (UTC+7) — both logs key off a **local** `entryDate` (`YYYY-MM-DD`).
3. For the trend cases, the account needs data on several days: either log them by hand via the date
   picker, or seed via API ([01 §5](../01-Test-Environment-Builds-and-Data.md#5-seeding-realistic-tracking-history)).

## ⚠️ Read before writing any defect: showcase vs. shipped build

| Showcase §4 says | The build under test does |
|---|---|
| Sleep: "Log bedtime, wake time, and how you slept (5 levels, from *7–9 hrs* great to *<3 hrs*)" | ✅ Matches — a date, a bedtime, a wake time and 5 quality levels (5 = excellent → 1 = terrible) |
| Nutrition: "Log meals (breakfast/lunch/dinner/snack)" | ⚠️ The screen logs a **meal count** (a stepper), not per-meal-type entries. Meal-type labels (*Sáng/Trưa/Tối/Ăn vặt*) exist in the translation file but the save payload carries no meal type. |
| Nutrition: "your water intake in litres" | ✅ Matches on screen — but note the wire field is `waterGlasses` and holds **millilitres** (`litres × 1000`). See `M04-06`. |
| Nutrition: "how full you felt" | ⚠️ The `satietyLevel` field is repurposed to carry the **meal count** as a string. The five satiety levels (*Năng lượng / Bình thường / Nuông chiều / Quá đà / Bỏ bữa*) exist as constants and translations but are not what the save path writes. |
| Nutrition: "even mindful eating (*Ăn chánh niệm*)" | ⚠️ A label exists; no corresponding field is sent. |

**Do not raise the three ⚠️ rows as new defects** — they are documentation-vs-implementation gaps, and
`M04-09` exists specifically to confirm and record them. Raise them as a **documentation** action:
either the showcase is trimmed to what ships, or the fields are implemented. Either is fine; silently
demoing a claim the code does not support is not.

---

## Test cases

### `E2E-M04-01` — Log a night's sleep
**Priority** P1 · **Type** Happy path · **Est.** 5 min

**Test data** — Date: today · Bedtime: `22:30` (previous evening) · Wake: `06:45` · Quality: **5** (excellent).

| # | Action | Expected result |
|---|---|---|
| 1 | Open *"Giấc ngủ"* from the home dashboard | `SleepMainScreen` loads with today selected and a 7-day chart area |
| 2 | Open the date control | A date picker appears; today is selected |
| 3 | Set the bedtime | The picker defaults sensibly (the app pre-fills the **previous** day at 22:00, which is the correct semantic for a night's sleep) |
| 4 | Set the wake time to `06:45` | Accepted |
| 5 | Choose quality level **5** | The option highlights, with its title and subtitle text visible |
| 6 | Save | Success feedback; the screen returns to its overview state showing today's entry |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/sleeps/{profileId}" \
  -H "Authorization: Bearer $TOKEN" | jq '.[0]'
```
Bedtime, wake time and `quality: 5` match; the entry date is today's **local** date.

**Pass criteria** — The sleep log persists with the exact times entered and the chosen quality.

---

### `E2E-M04-02` — All five sleep-quality levels are selectable and distinct
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | For five different dates, log quality 5, 4, 3, 2 and 1 in turn | Each saves |
| 2 | Observe each option in the picker | Each level has its **own** colour and face icon (happy → cry) and its own title + subtitle copy |
| 3 | Check the 7-day chart | Each day plots at a visibly different height/colour matching its level |
| 4 | Verify the backend values | `quality` values are the integers 5…1 — not strings, not an off-by-one |

**Pass criteria** — All five levels are reachable, visually distinct, and stored as the correct
integer.

---

### `E2E-M04-03` — Bedtime after wake time (overnight crossing)
**Priority** P1 · **Type** Edge · **Est.** 5 min

Sleep almost always crosses midnight, so "bedtime 22:30, wake 06:45" is the *normal* case, not the
edge one. The real edges are below.

| # | Action | Expected result |
|---|---|---|
| 1 | Log bedtime `23:50`, wake `00:20` (30 minutes) | Saves; any derived duration shown is **30 minutes**, not 23.5 hours |
| 2 | Log bedtime `13:00`, wake `15:00` (a nap, same day) | Saves; duration is 2 hours and is not treated as negative |
| 3 | Log a bedtime and wake time that are **identical** | Either rejected with a message or stored as zero — record which; it must not produce a negative or 24-hour duration |
| 4 | Check the 7-day chart after these entries | No spikes to absurd values; the chart remains readable |

**Pass criteria** — Durations derived from a midnight crossing are correct, and no entry produces a
negative or nonsensical duration in the chart.

---

### `E2E-M04-04` — Log nutrition: water and meals
**Priority** P1 · **Type** Happy path · **Est.** 5 min

**Test data** — Water `1.50` L · Meals `3` · Description `QA M04 — cơm, phở, và một ly trà sữa.`

| # | Action | Expected result |
|---|---|---|
| 1 | Open *"Dinh dưỡng"* (`Nhật ký dinh dưỡng`) | Screen shows the selected date, a 7-day meal chart, a 7-day water chart, and the quick-entry card *"Ghi nhanh"* |
| 2 | Use the water stepper (+ / −) to reach `1.50` | The value updates in 2-decimal litres with the unit **L** |
| 3 | Use the meal stepper to reach `3` | Shows `3` with the unit *"bữa"* |
| 4 | Enter the description in *"Mô tả & cảm xúc (tùy chọn)"* | Accepted; a counter shows `n/300` |
| 5 | Save (*"Lưu dinh dưỡng"*) | Success; a celebration sheet may appear for completing the nutrition category |
| 6 | Observe the charts | Today's bars update for both meals and water |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/foods/{profileId}" \
  -H "Authorization: Bearer $TOKEN" | jq '.[0]'
```
Expect `waterGlasses: 1500` (millilitres), `satietyLevel: "3"` (the meal count as a string),
`foodDescription` matching, and `entryDate` = today.

**Pass criteria** — Water and meal count round-trip correctly through their (oddly named) wire fields
and appear on both charts.

---

### `E2E-M04-05` — Save is blocked at zero meals
**Priority** P2 · **Type** Negative · **Est.** 3 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open nutrition on a date with no entry; leave meals at `0` | The save control is **disabled** |
| 2 | Add water only, still 0 meals | Save remains disabled |
| 3 | Increase meals to `1` | Save becomes enabled |
| 4 | Try to decrement meals below `0` | Not possible — no negative counts |

**Pass criteria** — A zero-meal log cannot be saved, and the disabled state is visible rather than a
silent no-op on tap.

**Note** — Record whether "I ate nothing today" is expressible. It currently is not, even though the
satiety constants include *"Bỏ bữa"* (skipped meals) — and skipped meals are exactly the signal
showcase §11 says a friend should be able to notice. Worth flagging as a **product** observation,
not a code defect.

---

### `E2E-M04-06` — Water unit round-trip precision
**Priority** P2 · **Type** Edge · **Est.** 4 min

The app stores `litres × 1000` rounded to an integer and divides by 1000 on read.

| # | Action | Expected result |
|---|---|---|
| 1 | Set water to `0.25` L and save | Backend shows `waterGlasses: 250` |
| 2 | Reopen the screen for that date | The stepper reads back exactly `0.25` |
| 3 | Set a large value (e.g. `9.99` L) and save | Round-trips exactly; the chart scales rather than clipping |
| 4 | Set water to `0` with meals ≥ 1 and save | Saves; reads back as `0.00`, not blank or `NaN` |

**Pass criteria** — Every value entered reads back identically after a reload. No drift, no `NaN`.

---

### `E2E-M04-07` — 7-day trends are correct and aligned to weekdays
**Priority** P1 · **Type** Happy path · **Est.** 6 min

**Preconditions** — Sleep and nutrition data on at least 4 of the last 7 days, with **different**
values so mis-ordering is visible.

| # | Action | Expected result |
|---|---|---|
| 1 | Open the sleep chart | Seven columns labelled `Hai Ba Tư Năm Sáu Bảy CN` (Mon…Sun) |
| 2 | Cross-check each plotted day against the raw API values | Every plotted value matches its date — **no off-by-one weekday shift** |
| 3 | Check a day with **no** data | Renders as a gap/zero, clearly distinguishable from a logged zero |
| 4 | Repeat for both nutrition charts (meal count, water) | Same correctness |
| 5 | Return to Home | The sleep and nutrition mini-cards show today's values consistent with the detail screens |

**Pass criteria** — The chart's day *n* equals the API's day *n* for all seven columns, on both
screens.

**Note** — Weekday alignment is the classic bug here: `Date.getDay()` returns 0 for Sunday while the
label array starts at Monday. Check the **first and last** columns explicitly; a shift is invisible in
the middle of a flat week.

---

### `E2E-M04-08` — One log per day: re-saving updates rather than duplicates
**Priority** P1 · **Type** Edge · **Est.** 5 min

The nutrition screen keeps a single log per date and switches its button to *"Cập nhật dinh dưỡng"*
when one exists.

| # | Action | Expected result |
|---|---|---|
| 1 | Save a nutrition log for today (meals 2, water 1.0) | Created |
| 2 | Stay on the screen, change to meals 4, water 2.0, save again | The button reads *update*, not *create* |
| 3 | Query the backend for today | **One** record, with the updated values — not two |
| 4 | Leave the screen, come back, save again with different values | Still one record, updated |
| 5 | Repeat the equivalent on the sleep screen for the same date | Record whether sleep also enforces one-per-day or appends — **either is acceptable, but the charts must not double-count** |

**Pass criteria** — Nutrition holds exactly one record per date and updates it in place; no chart
double-counts a re-saved day.

---

### `E2E-M04-09` — Confirm the documented feature gaps (meal types, satiety, mindful eating)
**Priority** P2 · **Type** Negative · **Est.** 5 min

This case exists to make the gap table above **verified fact on this build**, not an assumption.

| # | Action | Expected result |
|---|---|---|
| 1 | Look for a control to tag a meal as *Sáng / Trưa / Tối / Ăn vặt* | Record: present and functional, present but inert, or absent |
| 2 | Look for a control to record how full you felt (*Năng lượng … Bỏ bữa*) | Record the same |
| 3 | Look for *"Ăn chánh niệm"* (mindful eating) | Record the same |
| 4 | Save a log with anything found in 1–3 set | Inspect the backend payload: confirm whether the value is transmitted |
| 5 | Write the findings into the run log | The showcase and the build now have a documented, dated comparison |

**Pass criteria** — This case cannot "fail". It **must** be executed and its findings recorded before
the demo, so nobody claims a feature on stage that the payload does not carry.

---

## Exit criteria for M04

- `M04-01`, `M04-04`, `M04-08` pass — both write paths are correct and non-duplicating.
- `M04-07` passes — trends are correctly aligned (this is what the council sees).
- `M04-09` is **executed and recorded**, and any gap it confirms is reflected in the showcase text or
  raised as future work.
- No **S1** defect open against sleep or nutrition persistence.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| Wire field `waterGlasses` actually holds millilitres | ✅ Known naming quirk — verify the round-trip, not the name |
| `satietyLevel` carries the meal count as a string | ⚠️ Documented gap — see the table above and `M04-09` |
| No per-meal-type or mindful-eating field is sent | ⚠️ Documented gap |
| Save disabled at 0 meals | ✅ By design (`isSaveDisabled`) |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
