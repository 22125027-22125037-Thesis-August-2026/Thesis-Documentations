# E2E-M02 — Emotional Journal "Góc tâm tư"

> **Showcase §2** — *"a corner for your thoughts" — your private feelings diary.*
> **Plan priority: P0.** The journal is the richest thing a user creates in uMatter, the only feature
> with **media upload**, and the data a therapist most wants to read. A silent write failure here
> loses something the user cannot reconstruct.

| | |
|---|---|
| **Scope** | Create / edit / delete a journal entry, mood + tags + up to 5 photos, the 365-day mood calendar (*"Dòng cảm xúc"*), the journal streak, search and date filtering |
| **Out of scope** | Home-screen mood check-in (→ [M01](E2E-M01-Mood-Check-In-and-Focus-Mode.md)), therapist visibility of the journal (→ [M13](E2E-M13-Privacy-Data-Access-Grants.md)), trophy credit (→ [M11](E2E-M11-Daily-Trophy-and-Streaks.md)) |
| **Services** | Auth, Tracking (`/api/v1/tracking/diaries`), MinIO (photo storage) |
| **Screens** | `DiaryOverviewScreen`, `DiaryEntryScreen`, `TagSelector` |
| **Est. duration** | 50 min |
| **Cases** | 10 |

---

## Preconditions

1. TEEN-A logged in; `profileId` recorded.
2. **Photo library permission** granted, and the device gallery contains **at least 7 images** (needed
   to test the 5-attachment cap and the over-selection path).
3. At least one image in the gallery is **large** (> 5 MB) for `M02-04`.
4. Stable network — photo upload is the heaviest request the app makes.

## Test data

| Item | Value |
|---|---|
| Entry title | `QA M02 — ngày kiểm thử <dd/MM>` |
| Body | `Hôm nay mình thấy khá ổn. Đây là bài kiểm thử tự động hoá tay.` |
| Mood | **Vui vẻ** (→ `EXCELLENT`) for the happy-path entry |
| Tags | Any two from the tag selector |
| Photos | 1 for the basic case, 5 for the cap case |

> **Implementation note the tester must know:** tags are **not** a separate field server-side. The app
> serialises them into the entry's `content` as `Tags: a, b` followed by a blank line and the note,
> then parses them back on read. So a backend payload showing `content: "Tags: học tập, gia đình\n\n…"`
> is **correct**, not corruption. Round-tripping this is what `M02-05` tests.

---

## Test cases

### `E2E-M02-01` — Create a journal entry with title, mood and body
**Priority** P0 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open the journal (*"Góc tâm tư"*) | Overview screen loads: search bar, 🔥 streak widget, date-range controls, entry list |
| 2 | Tap the new-entry action | `DiaryEntryScreen` opens in create mode, all fields empty, today's date pre-selected |
| 3 | Pick the mood **Vui vẻ** | Emotion chip activates |
| 4 | Enter the title | Title accepted |
| 5 | Enter the body in *"Bạn đang nghĩ gì?"* | Multi-line text accepted; the field scrolls rather than clipping |
| 6 | Save | Loading state, then return to the overview |
| 7 | Observe the overview list | The new entry appears at the top with the correct title, mood colour and today's date |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/diaries/{profileId}" \
  -H "Authorization: Bearer $TOKEN" | jq '.[0] | {id, title, moodTag, content, createdAt}'
```
`title` matches, `moodTag` is `EXCELLENT`, `content` contains the body text.

**Pass criteria** — The entry is persisted server-side with the correct title, mood tag and body, and
is listed on the overview without a restart.

---

### `E2E-M02-02` — Attach photos to an entry
**Priority** P0 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Start a new entry and fill title + mood | — |
| 2 | Tap the photo attach control | System photo picker opens (multi-select allowed) |
| 3 | Select **1** photo | Picker closes; a thumbnail appears in the entry with a remove (×) affordance |
| 4 | Save | Upload progresses; save completes without an error alert |
| 5 | Reopen the saved entry | The photo renders from its stored URL — not a local `file://` path |
| 6 | Force-stop, relaunch, reopen the entry | The photo still renders (proves it is on the server, not device cache) |

**Backend verification** — The entry's payload contains an attachment with a `fileUrl` pointing at the
object store. Fetch that URL directly; it must return the image.

**Pass criteria** — A photo attached in the app survives a cold restart and is served from the backend.

**Note** — Step 6 is the whole point. A thumbnail rendered from the picker's local URI looks identical
until the app is reinstalled, at which point the user's photos are gone. That failure is **S1**.

---

### `E2E-M02-03` — The 5-photo cap is enforced, with a clear message
**Priority** P1 · **Type** Edge · **Est.** 5 min

The app caps attachments at **5** per entry.

| # | Action | Expected result |
|---|---|---|
| 1 | Start a new entry | — |
| 2 | Select exactly 5 photos in one go | All 5 thumbnails appear |
| 3 | Tap the attach control again | An alert explains the limit has been reached; the picker does **not** add more |
| 4 | Remove one thumbnail (×), then attach 1 more | Succeeds — the count is a live cap, not a one-time check |
| 5 | Start a fresh entry and select **7** photos in a single pick | The app accepts only the first 5 and shows a "truncated" message explaining that some were not added |
| 6 | Save the 5-photo entry | Saves successfully; all 5 render after reopening |

**Pass criteria** — Never more than 5 attachments, the user is told why, and over-selection truncates
gracefully instead of failing the save.

---

### `E2E-M02-04` — Large photo and slow upload
**Priority** P1 · **Type** Edge · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Attach the largest image in the gallery (> 5 MB) | Thumbnail appears |
| 2 | Save and watch closely | A loading state is shown for the duration — the UI is not frozen and the save button cannot be double-tapped into two entries |
| 3 | Verify the result | Exactly **one** entry created, with the image viewable |
| 4 | Repeat on a throttled connection (Android developer options → network throttling, or 3G) | Slower but still completes, or fails with a clear message — never a silent half-save |

**Backend verification** — Exactly one diary record; its attachment URL resolves.

**Pass criteria** — Large uploads either complete or fail loudly; they never create duplicates or
orphan an entry with a broken image.

---

### `E2E-M02-05` — Tags round-trip correctly
**Priority** P1 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Create an entry, select **two** tags in the tag selector | Both chips show as selected |
| 2 | Type a body below the tags | — |
| 3 | Save, then reopen the entry for editing | **Exactly** the same two tags are selected, and the body text shows **without** the `Tags: …` prefix leaking into it |
| 4 | Inspect the raw backend payload | `content` begins `Tags: <tag1>, <tag2>` then a blank line then the body — the documented serialisation |
| 5 | Edit: deselect one tag, save, reopen | One tag remains; body still intact |

**Pass criteria** — Tags survive a save/reopen cycle and never contaminate the visible body text.

**Note** — Because tags live inside `content`, a body that itself starts with the literal text
`Tags: ` is a genuine parser edge. Try it once: create an entry whose body's first line is
`Tags: không phải tag`. Record what happens — a mis-parse here is **S3** (cosmetic corruption of the
user's own words), and worth knowing before a demo.

---

### `E2E-M02-06` — The mood calendar colours days correctly
**Priority** P1 · **Type** Happy path · **Est.** 6 min

> Showcase §2: a mood calendar *"that colours each day positive / neutral / negative at a glance"*.

**Preconditions** — Entries exist on at least three different dates with different mood tones. Use the
entry screen's date picker to back-date, or seed via API
([01 §5](../01-Test-Environment-Builds-and-Data.md#5-seeding-realistic-tracking-history)).

| # | Action | Expected result |
|---|---|---|
| 1 | Create/ensure an entry with **Vui vẻ** (`EXCELLENT`, positive tone) on day A | — |
| 2 | Ensure an entry with **Ngạc nhiên** (`NEUTRAL`) on day B | — |
| 3 | Ensure an entry with **Buồn bã** (`BAD`, negative tone) on day C | — |
| 4 | Open the mood calendar (*"Dòng cảm xúc"*) | Days A, B, C are each marked, in **three visibly different** colours matching positive / neutral / negative |
| 5 | Check a day with no entry | Unmarked — no default colour that implies a mood the user never logged |
| 6 | Tap a marked day | Filters/navigates to that day's entry |
| 7 | Scroll back several months | The calendar navigates without error and shows the 365-day journey framing |

**Pass criteria** — Tone colouring matches the mood tag on every checked day, and empty days stay
empty.

**Note** — Verify the mapping specifically for `NEUTRAL`: neutral rendered in the negative colour is a
small bug with an outsized effect, because the calendar is the first thing a therapist or a parent
reads as a summary of the teen's month.

---

### `E2E-M02-07` — Journal streak counts consecutive days
**Priority** P1 · **Type** Happy path · **Est.** 5 min + multi-day

The 🔥 streak on the overview is computed **client-side from entry `createdAt` values**, so it depends
on the device's local date interpretation.

| # | Action | Expected result |
|---|---|---|
| 1 | With entries on today and yesterday, open the overview | Streak widget shows **2** with the label *"ngày"* |
| 2 | Add a second entry for **today** | Streak stays **2** — a streak counts days, not entries |
| 3 | Ensure an entry exists for the day before yesterday too | Streak shows **3** |
| 4 | Introduce a gap (a day with no entry, earlier in the range) | Streak counts only the unbroken run ending today |
| 5 | Cross a real midnight without journaling, then reopen | Streak resets or holds according to the app's rule — **record which**, since the showcase promises streaks "keep you motivated" and an unexplained reset is demoralising |

**Backend verification** — Compare the streak number against the raw `createdAt` dates from
`GET /api/v1/tracking/diaries/{profileId}`.

**Pass criteria** — The streak equals the length of the unbroken run of days with ≥1 entry, and does
not double-count multiple entries on one day.

---

### `E2E-M02-08` — Search and date-range filtering
**Priority** P2 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Type a distinctive word from one entry's title into the search box | The list narrows to matching entries as you type |
| 2 | Search a word that appears only in a **body**, not a title | Record whether body text is searched; either behaviour is acceptable but it must be **consistent** |
| 3 | Clear the search with the × control | Full list returns |
| 4 | Search a term with **Vietnamese diacritics** (`buồn`) | Matches; then try the unaccented form (`buon`) and record whether it matches — diacritic-insensitive search is a real usability point for Vietnamese users |
| 5 | Set a from/to date range covering only day A | Only day A's entries listed |
| 6 | Set a range with no entries | A clear empty state, not a blank screen or a spinner that never resolves |
| 7 | Combine search + date range | Both filters apply together (AND, not OR) |

**Pass criteria** — Filters narrow correctly, combine correctly, and clear correctly.

---

### `E2E-M02-09` — Edit and delete an entry
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open an existing entry for editing | All fields pre-populate: title, mood, tags, body, existing photos |
| 2 | Change the title and the mood, save | Overview reflects both changes immediately |
| 3 | Reopen — confirm changes stuck | Edited values shown |
| 4 | Remove an existing photo and save | Photo gone after reopening |
| 5 | Delete the entry | A confirmation is required before deletion |
| 6 | Confirm deletion | Entry disappears from the list and from the mood calendar for that day |
| 7 | Cold restart and check again | Entry is still gone — deleted server-side, not just locally |

**Backend verification** — `PUT /api/v1/tracking/diaries/{id}` reflected in a subsequent GET; after
delete, the id returns 404 / is absent from the collection.

**Pass criteria** — Edits and deletes persist server-side and update every view that showed the entry
(list **and** calendar).

---

### `E2E-M02-10` — Entry creation with no network
**Priority** P1 · **Type** Recovery · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Compose a full entry **with a photo** while online, but enable aeroplane mode before saving | — |
| 2 | Save | A clear failure message; the composed content is **not** discarded — the user can retry without retyping |
| 3 | Restore network and retry save | Succeeds |
| 4 | Verify backend | Exactly **one** entry, with the photo |
| 5 | Check the overview | No ghost/duplicate entry from the failed attempt |

**Pass criteria** — An offline save fails visibly, preserves the draft, and the retry produces exactly
one entry.

**Note** — Losing a long, emotional journal entry to a network blip is the worst failure in this plan
from a user's point of view, even though no data is *corrupted*. Rate a discarded draft **S2** at
minimum.

---

## Exit criteria for M02

- `M02-01`, `M02-02`, `M02-09`, `M02-10` pass — the full create/read/update/delete cycle is durable.
- `M02-03` and `M02-05` pass — attachment cap and tag round-trip behave.
- `M02-06` passes — calendar tone colours are correct (this is the screen shown in the demo).
- No **S1** defect open against journal persistence or photo storage.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| `content` on the wire starts with `Tags: …` | ✅ **By design** — tags are serialised into content and parsed on read |
| The streak is computed on-device | ✅ By design — not a server field; expect it to follow the device's local dates |
| Quick-note field caps at 100 characters | ✅ By design (`MAX_QUICK_NOTE_LENGTH`) — distinct from the main body, which is not capped |
| Only images can be attached | ✅ By design — the picker requests photos only |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
