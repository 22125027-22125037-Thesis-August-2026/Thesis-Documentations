# E2E-M01 — Daily Mood Check-In & Focus Mode Reminders

> **Showcase §1** — *"Check in with your feelings — and get gently reminded to."*
> **Plan priority: P0.** This is the feature the entire product is built around: every other feature
> (charts, journal, AI grounding, therapist context) consumes the mood log. If check-in fails,
> nothing downstream is trustworthy.

| | |
|---|---|
| **Scope** | Home-screen mood check-in, mood persistence, dashboard reflection, low-mood safety net, Focus Mode scheduling & delivery, Quiet Hours, schedule refill |
| **Out of scope** | Journal moods (→ [M02](E2E-M02-Emotional-Journal.md)), mood-driven AI grounding (→ [M07](E2E-M07-AI-Companion-Ban-Tam-Giao.md)), the trophy (→ [M11](E2E-M11-Daily-Trophy-and-Streaks.md)) |
| **Services** | Auth, Tracking (`/api/v1/tracking/moods`), Dashboard (`/api/v1/dashboard/summary`), Notifee (local, on-device) |
| **Screens** | `HomeScreen` → `MoodCheckInCard`, `LowMoodPromptSheet`, `ProfileScreen` → `FocusModeToggle` |
| **Est. duration** | 60 min active + one overnight window |
| **Cases** | 10 |

---

## Preconditions

1. Release build installed; `HARDCODED_TEST_TOKEN` empty ([01 §2](../01-Test-Environment-Builds-and-Data.md#2-build-under-test)).
2. **TEEN-A** registered and logged in; `profileId` recorded.
3. Notification permission **granted**; battery optimisation **disabled** for uMatter.
4. Device timezone **ICT (UTC+7)**, clock automatic.
5. Current local time is **outside** quiet hours (i.e. between 07:00 and 22:00) when you start — several
   cases schedule reminders and you need the next slot to be a real one.
6. For `M01-05`: no low-mood prompt shown in the last 6 hours (fresh install, or an account that has
   not logged a low mood today).

## Test data

| Item | Value |
|---|---|
| Positive emotion | **Vui vẻ** (→ `moodTag: EXCELLENT`, score 10) |
| Neutral emotion | **Ngạc nhiên** (→ `NEUTRAL`, score 6) |
| Low emotion | **Buồn bã** (→ `BAD`, score 3) |
| Lowest emotion | **Tức giận** (→ `TERRIBLE`, score 1) |
| Note text | `QA M01 — kiểm thử cảm xúc <HH:mm>` |

> The card offers **eight Plutchik emotions** that map onto the five mood tags the showcase describes
> (`TERRIBLE / BAD / NEUTRAL / GOOD / EXCELLENT`). Cases below name the emotion the tester taps and
> the tag it must produce — **verifying that mapping is part of the test**, since the AI, the calendar
> colouring and the therapist view all key off the tag, not the emotion.

---

## Test cases

### `E2E-M01-01` — Log a mood with a note from the home screen
**Priority** P0 · **Type** Happy path · **Est.** 4 min

**Preconditions** — Logged in as TEEN-A, on the Home tab.
**Test data** — Emotion **Vui vẻ**; note `QA M01 — kiểm thử cảm xúc 10:15`.

| # | Action | Expected result |
|---|---|---|
| 1 | Observe the mood card below the hero header | Card titled *"Cảm xúc lúc này"* is visible with a grid of emotion chips; no chip is pre-selected if today has no entry |
| 2 | Tap the **Vui vẻ** chip | Chip becomes visually active (filled icon, raised circle); a note input appears with the placeholder `Ghi chú cho "Vui vẻ" (tuỳ chọn)...` |
| 3 | Type the note text | Text accepted, no truncation on screen |
| 4 | Tap the save/confirm control | Brief loading state, then the card switches to its "logged" state showing the chosen emotion; no error alert |
| 5 | Observe the rest of the home screen | The mood-related mini-dashboard refreshes without a manual pull |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/moods/{profileId}" \
  -H "Authorization: Bearer $TOKEN" | jq '.[0]'
```
The newest record must have `moodTag: "EXCELLENT"`, `positivityScore: 10`, `content` equal to the typed
note, and a timestamp within a minute of the tap.

**Pass criteria** — The mood is persisted server-side with the correct tag **and** score, and the home
screen reflects it without a restart.

---

### `E2E-M01-02` — Save without a note (default content is generated)
**Priority** P1 · **Type** Edge · **Est.** 3 min

**Preconditions** — M01-01 done (a mood already exists today).
**Test data** — Emotion **Ngạc nhiên**, note left empty.

| # | Action | Expected result |
|---|---|---|
| 1 | Tap **Ngạc nhiên** | Chip activates, note field appears empty |
| 2 | Save without typing anything | Save succeeds — the note is **not** a required field |
| 3 | Check the saved entry in the app | The entry exists for today |

**Backend verification** — The newest mood record has `moodTag: "NEUTRAL"`, `positivityScore: 6`, and
`content` auto-filled as `Cảm xúc hôm nay: Ngạc nhiên` (the app substitutes a default rather than
sending an empty string).

**Pass criteria** — An empty note produces a valid record with the generated default content, not a
validation error and not an empty `content`.

---

### `E2E-M01-03` — Multiple check-ins in one day all persist
**Priority** P0 · **Type** Happy path · **Est.** 5 min

The showcase promises a nudge "roughly every 90 minutes", which only makes sense if the day can hold
many check-ins. This case proves a second check-in **adds** a record rather than overwriting.

| # | Action | Expected result |
|---|---|---|
| 1 | Note the current count of today's mood records (backend) | Record the number *n* |
| 2 | Log **Tin tưởng** with a note | Save succeeds |
| 3 | Wait ~1 min, log **Lo lắng** with a different note | Save succeeds |
| 4 | Re-query the backend | Count is *n + 2*; both notes present, distinct timestamps |
| 5 | Return to Home | The card shows the **most recent** mood (Lo lắng), not the first |

**Pass criteria** — Each check-in creates its own record; the card reflects the latest.

**Note** — If the second save silently replaces the first, that is **S2** and it breaks Focus Mode's
entire premise. Check the timestamps, not just the count.

---

### `E2E-M01-04` — Mood survives a cold restart and reaches the dashboard
**Priority** P0 · **Type** Persistence · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Force-stop the app (swipe from recents + force stop in settings) | App fully terminated |
| 2 | Relaunch and land on Home | The card shows today's latest mood **after** load — sourced from the API, not stale local state |
| 3 | Pull to refresh | Refresh completes, values unchanged |
| 4 | Query the dashboard summary | Today's mood is reflected in the summary payload |

**Backend verification**
```bash
curl -s https://umatter-apcs.duckdns.org/api/v1/dashboard/summary \
  -H "Authorization: Bearer $TOKEN" | jq
```

**Pass criteria** — After a cold start, the displayed mood comes from the server and matches
`M01-03` step 5. A blank card after restart means the write never landed (S1) or the read is broken (S2).

---

### `E2E-M01-05` — Low mood triggers the caring safety-net prompt
**Priority** P0 · **Type** Happy path · **Est.** 5 min

> Showcase §1: *"when you log a low mood, uMatter responds with care — offering a calming prompt or a
> path to support rather than leaving you alone with it."*

**Preconditions** — **No low-mood prompt shown in the last 6 hours.** The app rate-limits this sheet
via `@low_mood_prompt_last_shown` with a 6-hour cooldown. If in doubt, reinstall.

| # | Action | Expected result |
|---|---|---|
| 1 | Log **Buồn bã** (→ `BAD`) with a note | Mood saves normally |
| 2 | Observe immediately after save | The low-mood sheet (`LowMoodPromptSheet`) slides up **without** the user asking |
| 3 | Read its content | Offers a calming action and/or a route to support; tone is supportive, not alarming; no crisis-level language for a merely low mood |
| 4 | Follow each route offered (breathing, AI companion, support area) | Each navigates to the correct screen and back returns to Home |
| 5 | Dismiss the sheet | Sheet closes; Home is interactive |
| 6 | Immediately log **Tức giận** (→ `TERRIBLE`) | Mood saves, but the sheet does **not** reappear — the 6-hour cooldown suppresses it |

**Backend verification** — Both moods are persisted (`BAD` then `TERRIBLE`); the suppression is a
client-side anti-nag rule and must not suppress the *write*.

**Pass criteria** — The sheet appears for the first low/terrible mood, offers a working path to
support, and is rate-limited on the second — without ever blocking the mood from being saved.

---

### `E2E-M01-06` — Enable Focus Mode and confirm a schedule is created
**Priority** P1 · **Type** Happy path · **Est.** 5 min

**Preconditions** — Notification permission granted. Current time between 07:00 and 20:00 so the next
slot lands today.

| # | Action | Expected result |
|---|---|---|
| 1 | Go to Profile tab and find the Focus Mode row (*"Chế độ tập trung"*) | Row shows a bell icon, a title and an "off" subtitle; switch is off |
| 2 | Toggle the switch on | Brief spinner while the schedule is built, then the switch stays on |
| 3 | Observe the row after enabling | Subtitle changes to the "on" variant and a **quiet-hours badge** appears reading the window as `22`–`07` |
| 4 | Leave Profile and return | Switch is still on (state read from `@focus_mode_enabled`) |
| 5 | Force-stop and relaunch, return to Profile | Switch is still on — the setting is persisted, not in-memory |

**Device verification** — Notifications are scheduled locally via Notifee, so there is no server-side
artefact. Confirm scheduling happened by observing delivery in `M01-07`, and (optionally) with
`adb logcat` while toggling.

**Pass criteria** — Enabling produces a persisted "on" state with the quiet-hours badge, and survives
a cold restart.

---

### `E2E-M01-07` — A Focus Mode nudge actually arrives, and deep-links correctly ⏰ *long-wait case*
**Priority** P1 · **Type** Happy path · **Est.** 5 min active, up to 95 min wait

The scheduler places the first slot **90 minutes** after enabling, skipping quiet hours, and creates
**both** nudge types at *every* slot — so each slot delivers **two** notifications at the same instant.

| # | Action | Expected result |
|---|---|---|
| 1 | Enable Focus Mode at a recorded time `T` (see M01-06) | Schedule created |
| 2 | Use the phone normally; do **not** force-stop the app | — |
| 3 | At `T + 90 min` (± a few minutes) check the notification shade | **Two** uMatter notifications have arrived together, on the channel *"Chế độ tập trung"* |
| 4 | Read both | ① title **"Nhắc nhở nhẹ 💚"**, body *"Nhắc nhở nhẹ: Hãy uống một ngụm nước và hít thở sâu nhé!"* · ② title **"Cập nhật cảm xúc 💙"**, body *"Đừng quên ghi lại cảm xúc của bạn hôm nay nhé!"* |
| 5 | With the app in the **foreground**, tap the *Cập nhật cảm xúc* one | App navigates to the **new diary entry** screen (`DiaryEntry`) — it carries `target: 'diary-new'` |
| 6 | Force-stop the app, wait for the next slot, tap the same nudge from a cold start | App launches and *still* lands on `DiaryEntry` — the pending target is stored and replayed once navigation is ready |
| 7 | Tap the *Nhắc nhở nhẹ* one | App opens to wherever it was; no deep link is attached to this nudge (by design) |
| 8 | Wait for the following slot (`T + 180 min`) | Another pair arrives |

**Pass criteria** — Both nudges are delivered within ±10 minutes of the expected slot, are readable in
Vietnamese, and the mood nudge opens the diary-entry screen from **both** foreground and cold start.

**⚠️ Judgement call to record, not a pass/fail** — Two simultaneous notifications every 90 minutes is
**twice** what showcase §1 describes ("a gentle nudge roughly every 90 minutes") and works against the
"never nagging" promise. Verify the observed behaviour and log it; whether to alternate the two types
instead of pairing them is a product decision, and worth raising as **S3** if the pair feels noisy in
practice.

**Failure triage before filing a defect**
- Was battery optimisation disabled? OEM power management (Xiaomi, Oppo, Samsung) kills scheduled
  work aggressively — retest with it off before reporting.
- Was the app force-stopped? A force-stopped Android app cannot deliver scheduled notifications.
  (Step 6 stops it *after* the notification is already scheduled, which is a different thing.)
- Was the slot inside quiet hours? Then absence is **correct** — that is `M01-08`.

---

### `E2E-M01-08` — Quiet Hours are respected (no nudge 22:00–07:00)
**Priority** P1 · **Type** Edge · **Est.** 5 min active, overnight wait

> The showcase makes this a *care* promise: *"it never disturbs you while you sleep."* A single
> 03:00 buzz breaks a claim the app makes about itself.

| # | Action | Expected result |
|---|---|---|
| 1 | With Focus Mode on, confirm the badge shows the window `22`–`07` | Default quiet hours active |
| 2 | Leave the device on overnight, undisturbed, sound on | — |
| 3 | Next morning, review the full notification history (Android → Settings → Notifications → history) | **Zero** uMatter nudges with a timestamp between 22:00 and 07:00 |
| 4 | Check for a nudge shortly after 07:00 | The schedule resumes — the first post-quiet slot fires in the morning |

**Pass criteria** — No nudge inside the quiet window, and the schedule resumes afterwards (silence
that never ends is a different bug from silence that is respected).

---

### `E2E-M01-09` — The schedule refills itself, and cancelling really cancels
**Priority** P2 · **Type** Edge · **Est.** 10 min active, spread over ~3 days

The scheduler builds a bounded look-ahead: a **7-day horizon capped at 30 notifications per type**.
At a 90-minute interval with quiet hours excluded, that cap is reached in roughly **three days**, not
seven. Top-up happens **on app launch** — `rescheduleFocusModeIfNeeded()` re-builds the whole schedule
once fewer than **6** of the stored triggers are still pending. The showcase's claim is that reminders
*"quietly refill themselves so you stay supported"*.

| # | Action | Expected result |
|---|---|---|
| 1 | Enable Focus Mode and note the time | Schedule created |
| 2 | Reopen the app at least once a day for 3–4 days of normal use | — |
| 3 | On day 4, confirm nudges are still arriving | Delivery has **not** stopped after the initial ~30-slot batch was consumed — the launch-time top-up refilled it |
| 4 | **Now the negative half:** with Focus Mode still on, do not open the app for several days | If the batch drains to zero without a launch, delivery stops until the app is next opened |
| 5 | Toggle Focus Mode **off** | Switch turns off, no spinner error |
| 6 | Wait past the next expected slot | **No** nudge arrives — cancellation clears the whole pending batch, not just the next one |
| 7 | Toggle it back on | Nudges resume from a freshly built schedule |

**Pass criteria** — Reminders continue past the first batch for a user who opens the app daily, and
turning the feature off stops **all** pending nudges.

**Note** — Step 6 is the important half. A cancel that leaves orphaned scheduled notifications is
**S2**: the user turned the feature off and the app kept nagging.

**Known limitation to record (not a defect to file)** — Because the refill only runs on app launch, a
user who stops opening the app for ~3 days stops being reminded. That is precisely the disengaging
user the reminders exist for. Note it in the run log as a design observation; a background refresh
would be the fix, and it is legitimate future work rather than a broken promise.

---

### `E2E-M01-10` — Mood check-in with no network
**Priority** P1 · **Type** Recovery · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Enable aeroplane mode | Device offline |
| 2 | Attempt to log a mood | The app reports the failure clearly (an error state or alert) — it must **not** show a success state for a write that did not happen |
| 3 | Restore network | — |
| 4 | Retry the same check-in | Save succeeds |
| 5 | Verify backend | Exactly **one** record exists for that retry — no duplicate from the failed attempt |

**Pass criteria** — Offline failure is surfaced honestly and the retry produces exactly one record.

**Note** — A silent optimistic success followed by permanent data loss is **S1**: the user believes
their check-in is recorded and every downstream feature (AI grounding, therapist view) disagrees.

---

## Exit criteria for M01

- `M01-01` … `M01-05` and `M01-10` pass — the write path and the safety net are sound.
- `M01-06` passes and at least one nudge is observed in `M01-07`.
- `M01-08` shows zero notifications inside the quiet window.
- No **S1** defect open against mood persistence.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| Low-mood sheet does not reappear within 6 h | ✅ **By design** — anti-nag cooldown (`@low_mood_prompt_last_shown`) |
| No nudge while the app is force-stopped | ✅ Android platform behaviour, not a defect |
| Focus Mode has no server-side record | ✅ By design — scheduling is local (Notifee); it works offline and costs no push quota |
| The card offers 8 emotions but the API stores 5 tags | ✅ By design — emotions map to tags; verify the mapping, not the count |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
