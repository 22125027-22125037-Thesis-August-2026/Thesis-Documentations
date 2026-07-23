# E2E-M03 — Treasure Box "Hộp kho báu"

> **Showcase §3** — *"Cất giữ ảnh, lời nhắn và những điều khiến bạn thấy ấm lòng — mở ra mỗi khi cần
> được an ủi."*
> **Plan priority: P0.** The showcase calls this *"uMatter's most heartfelt feature"*, and §11 credits
> it to a named professional (Anh Vương Nguyễn Toàn Thiện, CEO of LUMOS). It will be demonstrated, and
> the psychological premise — *the memory is there when you are too low to recall it* — only holds if
> retrieval is **completely** reliable.

| | |
|---|---|
| **Scope** | Creating treasures across the 8 categories, image attachment, category filtering, the media viewer (image / video / audio), retrieval after reinstall, the low-mood entry path |
| **Out of scope** | The low-mood prompt that routes here (→ [M01-05](E2E-M01-Mood-Check-In-and-Focus-Mode.md)), crisis flows (→ [M12](E2E-M12-Crisis-Safety-and-Emergency-Support.md)) |
| **Services** | Auth, Tracking (`/api/v1/tracking/treasures`), MinIO |
| **Screens** | `TreasureBoxScreen`, `TreasureMediaViewer` |
| **Est. duration** | 35 min |
| **Cases** | 8 |

---

## Preconditions

1. TEEN-A logged in; `profileId` recorded.
2. Photo library permission granted; gallery contains at least one image **under** 25 MB and, for
   `M03-05`, one **over** 25 MB.
3. Network available (media uploads to object storage).

## The eight categories

The box is organised by category; each has an emoji, a colour and a label. All eight must be
creatable and filterable:

| id | Theme |
|---|---|
| `reasons` | Reasons to keep going |
| `joy` | Things that spark joy |
| `people` | People who matter |
| `dreams` | Dreams and hopes |
| `moments` | Moments captured |
| `affirmations` | Kind words received |
| `wins` | Small victories |
| `comfort` | Comfort (music/sound) |

> **What users can create vs. what the box can display.** The create flow attaches **images only**
> (single selection, ≤ 25 MB). The viewer additionally renders `VIDEO` and `AUDIO` treasures — those
> arrive as seeded/demo content, labelled *"Xem video"* and *"Nghe đoạn ghi âm"*. Test both: creation
> for images, playback for all three types.

---

## Test cases

### `E2E-M03-01` — Create a text-only treasure
**Priority** P0 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open the Treasure Box | Screen loads with the category filter row and existing treasures (or a warm empty state) |
| 2 | Tap add | The composer opens showing a grid of all **8** categories |
| 3 | Select the category **`affirmations`** | The option highlights in that category's colour and its emoji is adopted as the treasure's emoji |
| 4 | Enter a title and a message | Text accepted |
| 5 | Save without attaching media | Saves — media is optional |
| 6 | Observe the box | The new treasure appears with the right emoji and category colour |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/tracking/treasures/" \
  -H "Authorization: Bearer $TOKEN" | jq '.[0]'
```
Category, title and content match; `mediaUrl` is null.

**Pass criteria** — A text-only treasure persists with the correct category.

---

### `E2E-M03-02` — Create a treasure with a photo
**Priority** P0 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Add a treasure in category `moments` | — |
| 2 | Tap the media control | The picker opens for **photos**, single selection |
| 3 | Choose an image | A preview appears with a remove (×) control |
| 4 | Remove it, then re-attach a different image | The preview updates — the picker is re-runnable, not one-shot |
| 5 | Save | Upload completes; the treasure appears with its image |
| 6 | Force-stop, relaunch, open the treasure | The image renders from the server URL |

**Backend verification** — the record has a `mediaUrl` and `mediaType`; fetching the URL returns the
image bytes.

**Pass criteria** — The photo is stored server-side and viewable after a cold restart.

---

### `E2E-M03-03` — All eight categories are creatable and filterable
**Priority** P1 · **Type** Happy path · **Est.** 8 min

| # | Action | Expected result |
|---|---|---|
| 1 | Create one treasure in **each** of the 8 categories (title can be `QA <category>`) | All 8 save successfully |
| 2 | With no filter applied, view the box | All 8 are listed |
| 3 | Tap each category chip in turn | Only that category's treasures are shown, and the active chip is visually distinct |
| 4 | Tap the active chip again / choose "all" | The full list returns |
| 5 | Check the `comfort` category specifically | It has a distinct presentation (the screen surfaces a comfort item prominently) and does not error when empty |

**Pass criteria** — Every category can hold a treasure and every filter returns exactly its own
members.

---

### `E2E-M03-04` — Open a treasure in the media viewer (all three media types)
**Priority** P0 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Tap an **image** treasure | The viewer opens full-screen with the image; close returns to the box |
| 2 | Tap a **video** treasure (seeded) | The label reads *"Xem video"*; the viewer plays it with working controls |
| 3 | Tap an **audio** treasure (seeded) | The label reads *"Nghe đoạn ghi âm"*; audio plays |
| 4 | Close the viewer mid-playback | Playback **stops** — audio must not continue behind the box screen |
| 5 | Tap a **text-only** treasure | It opens/expands readably; no broken media placeholder is shown |
| 6 | Lock the screen during audio playback, then unlock | Behaviour is sane (either paused or still playing) and the app does not crash |

**Pass criteria** — Each media type opens and plays, closing stops playback, and text-only treasures
never render an empty media frame.

**Note** — Step 4 matters more than it looks: a comfort audio clip that keeps playing after the user
closes the screen, at 2 AM, is a genuinely bad experience for this feature's audience.

---

### `E2E-M03-05` — Oversized media is rejected with a clear message
**Priority** P1 · **Type** Negative · **Est.** 4 min

The composer rejects media above **25 MB**.

| # | Action | Expected result |
|---|---|---|
| 1 | Attempt to attach an image larger than 25 MB | An alert explains the size limit; the file is **not** attached |
| 2 | Confirm the composer state | Title/category/message the user already typed are **retained** — the rejection does not clear the draft |
| 3 | Attach a valid image instead and save | Saves normally |

**Pass criteria** — Oversized media is refused before upload, with a message, and without losing the
draft.

---

### `E2E-M03-06` — Treasures survive reinstall (the core promise)
**Priority** P0 · **Type** Persistence · **Est.** 6 min

> This is the case that validates the *feature's entire reason for existing*. A treasure box that
> lives only on the device is a photo album with extra steps.

| # | Action | Expected result |
|---|---|---|
| 1 | Note the exact contents of the box (count, categories, which have media) | Recorded |
| 2 | **Uninstall** the app completely | App and its data removed |
| 3 | Reinstall the same build and log in as TEEN-A | Login succeeds |
| 4 | Open the Treasure Box | **Every** treasure is present, in the right category, with its media rendering |
| 5 | Open a media treasure | Plays/renders from the server |

**Pass criteria** — 100 % of treasures and their media survive an uninstall/reinstall cycle. Anything
less is **S1**.

---

### `E2E-M03-07` — Delete a treasure
**Priority** P1 · **Type** Happy path · **Est.** 3 min

| # | Action | Expected result |
|---|---|---|
| 1 | Delete a treasure | A confirmation is required — this content is emotionally significant and must not be one-tap destructible |
| 2 | Confirm | It disappears from the box |
| 3 | Cold restart | Still gone |
| 4 | Verify the backend collection | The record is absent |
| 5 | Cancel a deletion instead | The treasure remains, untouched |

**Pass criteria** — Deletion requires confirmation, is server-side, and cancelling is safe.

---

### `E2E-M03-08` — Offline behaviour
**Priority** P1 · **Type** Recovery · **Est.** 4 min

The box's value is highest at the exact moment a user may have poor connectivity (in bed, on a bus).

| # | Action | Expected result |
|---|---|---|
| 1 | Open the box while online so content loads | Treasures visible |
| 2 | Enable aeroplane mode and reopen the box | Record the behaviour: cached content, a clear offline state, or an error — **any** is acceptable except a blank screen with no explanation |
| 3 | Attempt to create a treasure offline | Fails clearly; the draft is not silently discarded |
| 4 | Restore network and retry | Succeeds; exactly one record created |
| 5 | Open a media treasure offline | Either plays from cache or shows a clear "cannot load" state — never an infinite spinner |

**Pass criteria** — Offline states are explained rather than silently blank, and no duplicate is
created by the retry.

**Note** — Record the step-2 result explicitly in the run log. If the box is unreadable offline, that
is not a defect against current requirements, but it *is* a meaningful limitation of a feature sold as
*"open it whenever you need comforting"*, and is worth stating honestly in the thesis rather than
being discovered live.

---

## Exit criteria for M03

- `M03-01`, `M03-02`, `M03-04` and **`M03-06`** pass — creation, playback and durability.
- `M03-03` passes — all eight categories work.
- No **S1** defect open against treasure persistence or media retrieval.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| Users can attach images only; video/audio treasures come from seed data | ✅ By design in the current build — test creation for images, playback for all three |
| Media capped at 25 MB | ✅ By design (`MAX_MEDIA_BYTES`) |
| Only one media item per treasure | ✅ By design (`selectionLimit: 1`) — unlike the journal's 5 |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
