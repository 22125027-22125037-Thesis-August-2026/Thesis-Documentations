# E2E-M15 — First Run, Personalisation & Session Lifecycle

> **Showcase §10** — *"A friendly first-run tour walks you through every feature in under a minute (and
> you can replay it whenever you like). Bilingual — fully in Tiếng Việt and English. Your look —
> light, dark, or system theme. A home-screen widget keeps your day a glance away."*
> **Plan priority: P2** for the polish items — but the **session lifecycle** cases (`M15-08`, `M15-09`)
> are **P0**, because a token-refresh failure logs a user out mid-conversation and nothing else in the
> app can be trusted if authentication is unstable.

| | |
|---|---|
| **Scope** | Onboarding, registration & login, the first-run coach-mark tour and its replay, language, theme, the home-screen widget, static screens, token refresh, logout, and session persistence |
| **Out of scope** | Feature workflows (all other plans); the low-mood sheet (→ [M01](E2E-M01-Mood-Check-In-and-Focus-Mode.md)) |
| **Services** | Auth (`/api/v1/auth/*`), plus whatever the tour touches |
| **Screens** | `SplashScreen`, `OnboardingScreen`, `RegisterScreen`, `LoginScreen`, `TermsScreen`, the tour overlay, `ProfileScreen`, `FAQ`/`About`/`Contact`, the Android widget |
| **Est. duration** | 45 min |
| **Cases** | 10 |

---

## Preconditions

1. **A clean install** for `M15-01`–`M15-03`. Clearing app data is equivalent; logging out is **not**
   — the tour-seen flag and onboarding flag survive a logout.
2. A mailbox you can read, for registration.
3. Ability to leave the device idle long enough for an access token to expire (`M15-08`).

## ⚠️ Showcase vs. shipped build — verify before you assert

| Showcase §10 claims | What the build appears to do |
|---|---|
| *"A friendly first-run tour… you can replay it whenever you like"* | ✅ Implemented — a multi-step coach-mark tour spanning Home, the tab bar and Profile, gated by a seen-flag |
| *"Bilingual — fully in Tiếng Việt and English"* | ⚠️ **Both locale files are complete**, and `setAppLanguage()` exists — but **no screen calls it**. There appears to be no in-app language switcher. The i18n keys `profile.menuLanguage` / `profile.menuTheme` exist but are rendered nowhere. |
| *"Your look — light, dark, or system theme"* | ⚠️ The theme is a **single static palette** (`theme/colors.ts`); there is no theme context, no dark palette and no switcher |
| *"A home-screen widget"* | ✅ A widget bridge exists with mood caching and deep links |

`M15-04`–`M15-06` exist to **confirm or refute** these observations on the build under test. Record
the outcome; if confirmed, the showcase text should be trimmed to what ships (or the features
implemented) before the defence. Do not file them as code defects — they are scope/documentation
decisions for the team.

---

## Test cases

### `E2E-M15-01` — First launch: splash, onboarding, terms
**Priority** P2 · **Type** Happy path · **Est.** 5 min

**Preconditions** — Clean install, never launched.

| # | Action | Expected result |
|---|---|---|
| 1 | Launch the app | The splash appears, branded, and resolves — it does not hang |
| 2 | Observe what follows | The onboarding flow starts (first launch only) |
| 3 | Page through onboarding | Each screen renders in Vietnamese with correct imagery; forward/back work; a skip option is available |
| 4 | Complete or skip onboarding | You reach the auth screen |
| 5 | Open the Terms screen | Content renders and is scrollable to the end |
| 6 | Force-stop and relaunch | Onboarding does **not** repeat — the completion flag persisted |

**Pass criteria** — First launch shows onboarding exactly once and reaches the auth screen.

---

### `E2E-M15-02` — The first-run coach-mark tour
**Priority** P2 · **Type** Happy path · **Est.** 8 min

The tour auto-starts (once) for a **TEEN** account shortly after landing on Home, walks the home
screen, then the tab bar, then switches to the Profile tab.

| # | Action | Expected result |
|---|---|---|
| 1 | Log in as a brand-new TEEN account and wait on Home | The tour starts automatically after a brief delay |
| 2 | Step through it | Each step spotlights the right element: welcome → mood → dashboards → treasure → steps → breathing → nutrition → support → meditation → companion → find-therapist |
| 3 | Watch the scrolling | The screen **auto-scrolls** so each target is visible before the spotlight is drawn — the highlight never lands on an off-screen or misaligned element |
| 4 | Continue into the tab-bar steps | Each of the five tabs is spotlighted in turn |
| 5 | Continue past the tab steps | The tour **switches to the Profile tab** and highlights the streak trophy and daily trophy |
| 6 | Finish the tour | It closes cleanly and the app is fully interactive |
| 7 | Force-stop and relaunch | The tour does **not** auto-start again |
| 8 | Time the whole thing | Record it against the showcase's *"under a minute"* claim |

**Pass criteria** — Every step highlights the correct element, the tour completes, and it does not
re-trigger.

**Note** — Steps 3 and 5 are where coach-mark tours break: a spotlight measured before a scroll
settles lands in the wrong place, and a tab switch mid-tour can leave the overlay orphaned. Watch
both closely; a misaligned spotlight in a demo is very visible.

---

### `E2E-M15-03` — Register, log in, log out
**Priority** P0 · **Type** Happy path · **Est.** 6 min

The one auth case this folder keeps — not as field validation, but as the gate to everything else.

| # | Action | Expected result |
|---|---|---|
| 1 | Register a new TEEN account with a valid email and password | Success; you land in the app authenticated |
| 2 | Check `GET /api/v1/auth/me` with the stored token | Returns the new account with a `profileId` |
| 3 | Log out from Profile | A confirmation appears (*"Đăng xuất?" / "Hẹn gặp lại bạn sớm 💚"*) |
| 4 | Confirm logout | You return to the auth screen; tokens are cleared |
| 5 | Log back in with the same credentials | Succeeds; your data (mood, journal, treasures) is all still there |
| 6 | Try logging in with a **wrong** password | A clear error; no partial-auth state where the app looks logged in |
| 7 | Try a seeded `*.dev@mhsa.local` account | ⚠️ Expected to **fail** — the seed's BCrypt hash does not verify. This is known, not a defect |

**Pass criteria** — Register, login and logout all work, data survives a logout/login cycle, and bad
credentials fail cleanly.

---

### `E2E-M15-04` — Language: what is actually switchable
**Priority** P2 · **Type** Edge · **Est.** 6 min

| # | Action | Expected result / record |
|---|---|---|
| 1 | Search the Profile tab (and every settings surface) for a language control | **Record whether one exists** |
| 2 | If one exists: switch to English | The UI changes language immediately, without a restart |
| 3 | If none exists: set the **device** language to English and relaunch | Record whether the app follows the device locale |
| 4 | Either way, sample 10 screens in the non-Vietnamese state | Record any untranslated strings, and any hard-coded Vietnamese that never translates (e.g. the companion card's *"Bạn Tâm Giao"*, *"Online · Sẵn sàng lắng nghe"*, *"Bắt đầu ngay"*) |
| 5 | Check text overflow in the longer language | No clipped buttons or truncated labels |
| 6 | Restart and confirm the choice persisted | If switching is possible, `app_language` persists |

**Pass criteria** — This is a **finding-recorder**. It passes when the true state of language support
is documented with evidence.

**Expected finding** — Both locale files are complete and `setAppLanguage()` exists, but no screen
calls it, so there appears to be **no in-app language switcher**. Several strings are also hard-coded
in Vietnamese outside i18n. If confirmed, the showcase's *"fully in Tiếng Việt and English"* should be
softened, or a switcher added — it is a small change over infrastructure that already exists.

---

### `E2E-M15-05` — Vietnamese rendering quality
**Priority** P2 · **Type** Edge · **Est.** 5 min

Regardless of switching, Vietnamese is the demo language and must be flawless.

| # | Action | Expected result |
|---|---|---|
| 1 | Sample every main screen | All diacritics render correctly — no `?`, no boxes, no mojibake |
| 2 | Check the custom font on stacked diacritics (`ế`, `ữ`, `ỗ`) | Not clipped at the top of their line box |
| 3 | Enter Vietnamese text in every input (journal, chat, treasure, cancel reason) | Round-trips through the API unchanged |
| 4 | Check dates and numbers | Formatted for `vi-VN` (e.g. `23/7/2026`) |
| 5 | Check longer labels at **1.3× system font scale** | No clipping that hides a control |

**Pass criteria** — Vietnamese renders and round-trips correctly everywhere, including at larger font
scales.

---

### `E2E-M15-06` — Theme
**Priority** P2 · **Type** Edge · **Est.** 4 min

| # | Action | Expected result / record |
|---|---|---|
| 1 | Search for a theme/appearance control | Record whether one exists |
| 2 | Set the **device** to dark mode and relaunch | Record whether the app changes at all |
| 3 | Sample several screens in device dark mode | Record any unreadable combinations (e.g. system-coloured text on a light card) |
| 4 | Return the device to light mode | — |

**Pass criteria** — Finding-recorder. It passes when the app's true theme behaviour is documented.

**Expected finding** — The palette is a single static light theme with no dark variant and no
switcher, so showcase §10's *"light, dark, or system theme"* likely overstates the build. Step 3
matters even so: if any component picks up system colours while the rest stays light, a user in dark
mode may hit genuinely unreadable text — that **would** be a defect (**S3**), separate from the
missing feature.

---

### `E2E-M15-07` — Home-screen widget
**Priority** P2 · **Type** Happy path · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Add the uMatter widget to the Android home screen | It appears and renders |
| 2 | Log a mood in the app, then look at the widget | It reflects the latest mood (the app caches it for the widget on save) |
| 3 | Tap the widget | The app opens and deep-links to the right place — the mood/diary target opens the diary entry screen |
| 4 | Tap the widget while **logged out** | It routes to Login rather than a protected screen |
| 5 | Tap it from a **cold** (not running) state | The deep link is still honoured once the app is ready — the pending target is stored and replayed |
| 6 | Resize the widget if supported | Layout adapts without clipping |
| 7 | Log out, then check the widget | Record whether stale mood data from the previous user remains visible — a real privacy consideration on a shared phone |

**Pass criteria** — The widget shows current data and deep-links correctly from cold, warm and
logged-out states.

**Note** — Step 7 is worth attention: a home-screen widget is visible to anyone who picks up the
phone. If a previous user's mood persists after logout, that is **S2** on a device teenagers often
share with family.

---

### `E2E-M15-08` — Token refresh keeps the session alive
**Priority** P0 · **Type** Recovery · **Est.** 8 min

The API client refreshes an expired access token and retries the failed request, taking care never to
recurse on the auth endpoints themselves.

| # | Action | Expected result |
|---|---|---|
| 1 | Log in and note the time | — |
| 2 | Leave the app idle past the access-token lifetime (background it; check the configured TTL) | — |
| 3 | Return and perform an authenticated action (open the journal, send an AI message) | It **succeeds** — the client refreshed silently. The user is **not** bounced to the login screen |
| 4 | Watch logcat during step 3 | A `/auth/refresh` call, then the original request retried |
| 5 | Repeat with several actions in quick succession after expiry | Only **one** refresh occurs — parallel 401s do not trigger a refresh storm or duplicate writes |
| 6 | Perform a long action (photo upload) that spans the expiry | Completes after the refresh, without duplicating the upload |

**Pass criteria** — An expired access token is refreshed transparently and the user's action still
completes.

**Note** — Step 5 matters more than it looks. Home fires several parallel requests on focus; if each
401 triggers its own refresh, the refresh token can be rotated out from under the others and the user
is logged out **at random**. That presents as "the app randomly logs me out" and is **S1**.

---

### `E2E-M15-09` — Session termination is clean
**Priority** P0 · **Type** Recovery · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Invalidate the session server-side (or let the refresh token expire) and use the app | The user is logged out **gracefully** — a clear message, back to Login; no crash, no infinite retry loop |
| 2 | Confirm local cleanup | Access token, refresh token, role and `profileId` are all cleared |
| 3 | Confirm the FCM token is deregistered | See [M14-01](E2E-M14-Notifications-End-to-End.md) — the device stops receiving that account's pushes |
| 4 | Log out with **no network** | Local logout still completes — a failed server revoke must never trap the user in a signed-in state |
| 5 | Log in as a **different** account on the same device | No data from the previous account is visible anywhere: home dashboard, journal, treasures, notifications, widget |

**Pass criteria** — Logout always completes locally, clears all credentials, and leaks nothing to the
next account.

**Note** — Step 5 is a privacy test, not a convenience one. Any residue from account A visible to
account B on the same device is **S1**.

---

### `E2E-M15-10` — Static screens and app metadata
**Priority** P2 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Open **FAQ** | Content renders, scrolls, back works |
| 2 | Open **About** | Renders; check the version string matches the build under test |
| 3 | Open **Contact** | Renders; any links/emails work or are clearly informational |
| 4 | Check for the "support companion, not an emergency service" disclaimer | Present, per showcase §10, and it points to professional help |
| 5 | Check the app icon and name on the launcher | Correct branding, no placeholder |

**Pass criteria** — All static screens render with correct, current content and the disclaimer is
present.

---

## Exit criteria for M15

- **`M15-03`, `M15-08`, `M15-09` pass** — authentication is stable and leaks nothing between accounts.
- `M15-02` passes — the tour completes with correctly positioned spotlights.
- `M15-04` and `M15-06` executed and their findings recorded, so the showcase can be reconciled with
  the build before the defence.
- No **S1** defect open against session handling.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| Seeded `*.dev@mhsa.local` accounts cannot log in | ⚠️ Known — the seed's BCrypt hash does not verify |
| No in-app language switcher (`setAppLanguage` is uncalled) | ⚠️ Confirm in `M15-04`; reconcile with showcase §10 |
| No dark theme or theme switcher | ⚠️ Confirm in `M15-06`; reconcile with showcase §10 |
| Some strings are hard-coded in Vietnamese outside i18n | ⚠️ Record in `M15-04` |
| Onboarding and tour flags survive logout | ✅ By design — they are device-level, not account-level |

## Run log

| Date | Build / commit | Device + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
