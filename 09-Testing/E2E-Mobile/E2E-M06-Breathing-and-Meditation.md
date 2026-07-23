# E2E-M06 — Breathing & Meditation

> **Showcase §4** — *"🌬️ **Hít thở** — 'Một khoảnh khắc để tâm trí bình yên.' Guided breathing —
> inhale, hold, exhale — with calming music and reflective prompts"* and *"🎧 **Thiền & thư giãn** — a
> curated carousel of short sessions."*
> **Plan priority: P1.** Showcase §11 elevates breathing specifically: the counsellor consulted for
> the project called breath regulation *"one of the most evidence-backed ways to calm anxiety in the
> moment."* A breathing timer that drifts, or a session that fails to log, undercuts a claim the
> thesis makes on professional advice.

| | |
|---|---|
| **Scope** | The guided breathing session (phase timing, orb animation, 4 rounds, 5 reflection prompts, audio), session logging to the backend, the daily breathing goal, the meditation carousel and its video playback |
| **Out of scope** | Other tracking (→ [M04](E2E-M04-Sleep-and-Nutrition-Tracking.md), [M05](E2E-M05-Automatic-Step-Counting.md)), the low-mood route into breathing (→ [M01-05](E2E-M01-Mood-Check-In-and-Focus-Mode.md)) |
| **Services** | Tracking (`/api/v1/tracking/breathing`), YouTube (external, for meditation videos) |
| **Screens** | `BreathingExerciseScreen`, `BreathingAudio`, `MeditationCarousel` on Home |
| **Est. duration** | 40 min |
| **Cases** | 9 |

---

## Preconditions

1. TEEN-A logged in.
2. Device volume audible; not on silent (audio is part of the feature).
3. **Network available** — the meditation carousel plays YouTube content and will not work offline.
4. A stopwatch (a second phone) for `M06-02`.

## The session as built

| Element | Value |
|---|---|
| Phase pattern | **inhale 4 s → hold 4 s → exhale 6 s** (one cycle ≈ **14 s**) |
| Rounds before prompts | **4** (≈ 56 s of breathing) |
| Reflection prompts, in order | `goodMemory` → `recentAchievement` → `gratitude` → `personYouLove` → `safePlace` (**5**) |
| Daily goal | **300 s** (5 minutes) |
| Logged field | `durationSeconds` = elapsed seconds, upserted per day |
| Meditation carousel | 5 sessions incl. **4-7-8 breathing** (5 min), **body scan** (10 min), morning, sleep, calm-anxiety — each opens a YouTube video |

> ⚠️ **The meditation catalogue is explicitly a placeholder.** Its own source comment says the URLs are
> *"general guided-meditation videos chosen as defaults — swap them for curated/licensed content
> before release."* `M06-09` exists to check the content is at least appropriate and reachable, and to
> keep the licensing question visible.

---

## Test cases

### `E2E-M06-01` — Complete a full guided breathing session
**Priority** P1 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open *"Hít thở"* from the home dashboard | The breathing screen opens with a calm intro and a start control |
| 2 | Start the session | The orb begins animating and the phase label shows the inhale instruction |
| 3 | Follow through one full cycle | Labels cycle **inhale → hold → exhale** in that order, each with the orb expanding, holding, then contracting |
| 4 | Continue for all **4** rounds | After the 4th exhale the screen transitions to the reflection prompts |
| 5 | Read prompt 1 and advance | Prompts appear one at a time with a progress-dot indicator; the dots track position |
| 6 | Advance through all **5** prompts | The 5th prompt's action reads as a finish action, not "next" |
| 7 | Finish | A completion state appears; the screen returns cleanly |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/breathing/{profileId}" \
  -H "Authorization: Bearer $TOKEN" | jq '.[0]'
```
A record exists for today with a plausible `durationSeconds` (≥ 56 for four rounds, plus the time
spent on prompts).

**Pass criteria** — A complete session runs its 4 rounds and 5 prompts in order and writes one
breathing record for today.

---

### `E2E-M06-02` — Phase timing is accurate
**Priority** P1 · **Type** Edge · **Est.** 5 min

The therapeutic value depends on the *ratio*; a drifting timer teaches the wrong pattern.

| # | Action | Expected result |
|---|---|---|
| 1 | Start a session with a stopwatch running | — |
| 2 | Time the **inhale** phase | ≈ **4 s** (±0.5 s) |
| 3 | Time the **hold** phase | ≈ **4 s** (±0.5 s) |
| 4 | Time the **exhale** phase | ≈ **6 s** (±0.5 s) |
| 5 | Time a full cycle | ≈ **14 s** |
| 6 | Time all four rounds end-to-end | ≈ **56 s** — confirm there is **no cumulative drift**; round 4 should not be noticeably longer than round 1 |
| 7 | Confirm the orb animation matches the phase | The orb is fully expanded at the end of inhale and through hold, and contracted at the end of exhale — the visual and the label never contradict each other |

**Pass criteria** — Each phase is within ±0.5 s of its specification and four rounds complete in
≈ 56 s without drift.

---

### `E2E-M06-03` — Audio behaviour
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Start a session with volume up | Calming audio plays |
| 2 | Use the sound control on screen | Audio mutes/unmutes; the control's state matches what you hear |
| 3 | Leave the screen mid-session (back / home button) | **Audio stops.** It must not keep playing over the rest of the app or after the app is backgrounded |
| 4 | Return to the breathing screen | Either a fresh session or a resumed one — record which; both are acceptable if consistent |
| 5 | Start a session while other media (e.g. music) is playing | Behaviour is sane — either it takes audio focus or mixes; it must not produce distorted overlapping sound |
| 6 | Receive a phone call mid-session (or simulate an interruption) | Audio yields; the app does not crash |

**Pass criteria** — Audio starts, is controllable, and **always** stops when the user leaves the
screen.

**Note** — Step 3 is the one that matters. Background audio that outlives its screen is a common RN
media bug and is very visible in a live demo.

---

### `E2E-M06-04` — Abandoning a session part-way
**Priority** P1 · **Type** Edge · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Start a session, complete 2 of 4 rounds, then leave the screen | The session stops |
| 2 | Check the backend for a breathing record | Record whether a partial session is logged, and with what `durationSeconds` |
| 3 | Start again and abandon during the **prompts** phase | Same check |
| 4 | Complete a full session afterwards on the same day | Verify the daily record — since writes are an **upsert per day**, confirm whether the full session's duration **replaces** or **accumulates** with the earlier partial one |
| 5 | Record the semantics in the run log | — |

**Pass criteria** — Abandoning never crashes or writes a nonsensical duration (negative, zero-length,
or hours long). The accumulate-vs-replace semantics are documented after this case, whichever they
turn out to be.

**Note** — If a second session *replaces* rather than adds, a user who breathes three times a day sees
credit for only the last one, and the 5-minute daily goal becomes much harder to reach than intended.
That is worth an **S3** and a line in the run log.

---

### `E2E-M06-05` — The daily breathing goal (300 s)
**Priority** P2 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | With no breathing logged today, view the breathing summary | Progress toward the 5-minute goal reads 0 |
| 2 | Complete one session (~1.5 min) | Progress advances proportionally |
| 3 | Accumulate to ≥ 300 s (repeat sessions, or seed a day) | A goal-reached state appears |
| 4 | Exceed the goal | Progress clamps at 100 %; no overflow or negative "remaining" |

**Pass criteria** — Progress tracks the logged duration and clamps at the goal.

**Dependency** — This case's result depends on `M06-04`'s accumulate-vs-replace finding. Run M06-04
first.

---

### `E2E-M06-06` — Breathing history and 7-day range
**Priority** P2 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Ensure breathing logs exist on ≥ 3 different days | Seed via API if needed |
| 2 | View the breathing history / range view | Days appear with their durations, correctly dated |
| 3 | Cross-check against `GET /api/v1/tracking/breathing/{profileId}/range?from=&to=` | Screen values match the API exactly |
| 4 | Check a day with no session | Renders as empty/zero, distinguishable from a logged short session |

**Pass criteria** — Historical durations match the API for every displayed day.

---

### `E2E-M06-07` — The meditation carousel
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | On Home, find *"Thiền & thư giãn"* | A horizontally scrolling carousel |
| 2 | Scroll through it | **5** sessions are present: 4-7-8 breathing, body scan, morning, sleep, calm-anxiety |
| 3 | Check each card | Title, category and duration in minutes are shown; each has its own colour treatment; no missing/placeholder text |
| 4 | Tap a session | A video player opens (in-app player or modal) |
| 5 | Play it | Video plays with working controls; audio is audible |
| 6 | Close the player | Playback **stops**; you return to Home |
| 7 | Open a second session | Plays correctly — the player is reusable, not a one-shot |

**Pass criteria** — All 5 sessions are listed with complete metadata, and each plays and stops
cleanly.

---

### `E2E-M06-08` — Meditation and breathing without network
**Priority** P2 · **Type** Recovery · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Enable aeroplane mode; open the guided **breathing** session | The session **runs** — timing and animation are local; only the final log write needs the network |
| 2 | Complete the session offline | Completion does not crash; the log write fails quietly or is reported |
| 3 | Restore network, reopen the app, check the backend | Record whether the offline session was ever persisted — a lost session is acceptable-but-worth-noting; a crash is not |
| 4 | Offline, tap a **meditation** card | A clear "cannot load" state — **not** an infinite spinner or a blank black player |

**Pass criteria** — Breathing works offline as an experience; meditation fails with an explanation.

**Note** — Step 1 is a genuine strength worth confirming and stating: the single most useful feature
in an anxiety spike does not depend on connectivity.

---

### `E2E-M06-09` — Meditation content is appropriate and reachable
**Priority** P2 · **Type** Edge · **Est.** 5 min

The catalogue is a documented placeholder pointing at third-party YouTube URLs.

| # | Action | Expected result |
|---|---|---|
| 1 | Open each of the 5 videos in turn | All 5 load — no "video unavailable", no region block, no removed content |
| 2 | Watch the first ~30 s of each | Content matches its card's title/category (a "sleep" card should not open a workout video) |
| 3 | Check for advertising / unsuitable interruptions | Record whether ads play before a "calm anxiety" session — for this audience that is a real experience problem, not a nitpick |
| 4 | Check the language | Record whether content is Vietnamese or English — showcase §11 stresses a **Vietnamese-first** product, so English-only meditation content is a gap worth naming |
| 5 | Record findings | — |

**Pass criteria** — All 5 URLs resolve and are topically correct. Anything else is recorded as a
**content** action, not a code defect.

**Note** — This case is really a pre-release checklist item wearing a test case's clothes. Broken or
mismatched third-party links are the single most likely thing to embarrass a live demo, and they can
break at any time without a code change — so re-run this case the day before the defence.

---

## Exit criteria for M06

- `M06-01` and `M06-02` pass — the guided session runs correctly and to the specified timing.
- `M06-03` passes — audio always stops when the screen is left.
- `M06-07` passes — all 5 meditation sessions play.
- `M06-04`'s accumulate-vs-replace semantics are recorded.
- `M06-09` is re-run within 24 h of any demo.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| Meditation URLs are third-party placeholders | ✅ Documented in the source; a content/licensing task, not a defect |
| Breathing runs offline but cannot log | ✅ Expected — the timer is local, the write is not |
| Breathing writes are an upsert per day | ✅ By design — see `M06-04` for the consequence |
| Session is a fixed 4 rounds / 5 prompts, not configurable | ✅ By design in this build |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
