# E2E-M10 — Social: Friends & Real-Time Chat

> **Showcase §7** — *"uMatter has a friendly social layer: add friends by email, accept or decline
> requests, and chat in real time. Sometimes the best support is simply someone your own age who
> gets it."*
> **Plan priority: P1 — but test it early and carefully.** Real-time chat is the feature with the
> worst historical track record in this system: it **never established a session in production** until
> July 2026, because Spring's STOMP relay forwarded the client's `host` header to RabbitMQ as a
> virtual host and the broker rejected every CONNECT. The WebSocket upgrade succeeded the whole time,
> which is exactly why it went unnoticed. Test the STOMP layer, not the upgrade.

| | |
|---|---|
| **Scope** | Sending/accepting/rejecting friend requests by email, the friends & requests tabs, unread counts, opening a channel, real-time bidirectional messaging over STOMP, history, reconnection, unfriending |
| **Out of scope** | Therapist text consultations (→ [M09-05](E2E-M09-Consultation-Session-and-Aftercare.md)), sharing tracking data with a friend (→ [M13](E2E-M13-Privacy-Data-Access-Grants.md)) |
| **Services** | Auth, Social API (`/api/v1/social/*`), RabbitMQ via the STOMP relay at `/ws` |
| **Screens** | `MessageListScreen`, social `ChatScreen`, `FriendProfileScreen` |
| **Est. duration** | 50 min |
| **Cases** | 10 |

---

## Preconditions

1. **Two physical devices**, one signed in as TEEN-A and one as TEEN-B. This plan cannot be run on one
   device — logging out and back in tests message *history*, not *delivery*.
2. Both accounts' emails and `profileId`s recorded.
3. Both devices on network; be ready to toggle one to aeroplane mode (`M10-09`).
4. `adb logcat` available on at least one device, to read the STOMP connect line.

## The path under test

```
Phone (@stomp/stompjs, wss://umatter-apcs.duckdns.org/ws)
  → Caddy  (routes /ws straight to social-api — it does NOT pass through Nginx)
  → social-api (Spring STOMP broker relay, virtual host pinned to "/")
  → RabbitMQ
```

Destinations: subscribe `/user/queue/messages`, publish `/app/chat.send`. The JWT is sent in the
STOMP `CONNECT` headers, not just the HTTP handshake.

### ⚠️ Two things that will mislead you

1. **A successful WebSocket upgrade proves nothing.** The historical bug failed *after* a clean 101.
   The signal to trust is the app's own log line — `STOMP connected successfully` — and, server-side,
   `WebSocketMessageBrokerStats` reporting a non-zero session count.
2. **Curling `/ws` over HTTPS returns a misleading 400** unless you force HTTP/1.1 (`curl --http1.1`),
   because HTTP/2 forbids `Upgrade` headers. A 400 there is not evidence of a broken endpoint.

### ⚠️ There is no offline push for chat

Social's `message_sent` publish is commented out, and the `message.missed` consumer was **deleted** in
July 2026. **Messaging an offline friend produces no push notification.** That is the system's current
design — do not write a test case expecting one, and do not file it as a defect. It is recorded as a
known gap in the [README](../README.md#️-known-issues-every-tester-must-read-first).

---

## Test cases

### `E2E-M10-01` — Send a friend request by email
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | On TEEN-A's device, open the messages tab | Two tabs are visible: **friends** and **requests** |
| 2 | Start adding a friend | An email input appears |
| 3 | Enter TEEN-B's exact email and send | Success feedback; no error |
| 4 | On **TEEN-B's** device, open the requests tab | The request from TEEN-A is listed with their name/avatar and a request preview |
| 5 | Check the request badge/count | Reflects one pending request |

**Backend verification**
```bash
# as TEEN-B
curl -s "https://umatter-apcs.duckdns.org/api/v1/social/friends/requests" \
  -H "Authorization: Bearer $TOKEN_B" | jq
```

**Pass criteria** — A request sent by email on one device appears on the other's requests tab.

---

### `E2E-M10-02` — Friend-request negative paths
**Priority** P1 · **Type** Negative · **Est.** 6 min

| # | Action | Expected result |
|---|---|---|
| 1 | Send a request to an email that has **no account** | A clear message that the user was not found — not a silent success |
| 2 | Send a request to **your own** email | Rejected with an explanation |
| 3 | Send a **second** request to the same person while one is pending | Prevented or idempotent — not two pending requests |
| 4 | Send a request to someone who is **already** a friend | Prevented with a message |
| 5 | Enter a malformed email (`abc`) | Rejected before the network call, or handled cleanly |
| 6 | Enter the email with different capitalisation (`TEENB@…`) | Record whether it matches — case-sensitive lookup is a real usability trap |

**Pass criteria** — Every invalid case produces a specific, understandable message and creates no
stray request.

---

### `E2E-M10-03` — Accept a friend request
**Priority** P1 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | On TEEN-B, accept TEEN-A's request | The row shows a pending state during the call, then disappears from **requests** |
| 2 | Observe the tab | The app switches to the **friends** tab automatically |
| 3 | Check the friends list on TEEN-B | TEEN-A is present, with a chat channel available |
| 4 | Check the friends list on **TEEN-A** (refresh) | TEEN-B is present there too — the friendship is mutual |
| 5 | Verify server-side | A `DIRECT_FRIEND` channel exists via `GET /api/v1/social/chats/channels` on **both** accounts |

**Pass criteria** — Accepting creates a mutual friendship and a chat channel visible from both sides.

---

### `E2E-M10-04` — Reject a friend request
**Priority** P1 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Send a fresh request A→B | Appears on B |
| 2 | On B, reject it | Row disappears from requests |
| 3 | Check B's friends list | A is **not** present |
| 4 | Check A's friends list | B is **not** present, and A gets no misleading "accepted" state |
| 5 | Have A send another request afterwards | Allowed — rejection is not a permanent block (record if it is) |

**Pass criteria** — Rejection removes the request and creates no friendship on either side.

---

### `E2E-M10-05` — Real-time message delivery 🔑
**Priority** P1 · **Type** Happy path · **Est.** 8 min

The core case. Both devices open, side by side.

| # | Action | Expected result |
|---|---|---|
| 1 | On TEEN-A, open the chat channel with TEEN-B | Room opens; history (if any) loads |
| 2 | **Check logcat on A** | `STOMP connected successfully { brokerURL: wss://umatter-apcs.duckdns.org/ws }` appears — confirm this **before** interpreting any delivery result |
| 3 | On TEEN-B, open the same channel | Same connect line on B |
| 4 | Send `QA M10 — xin chào` from A | Appears immediately on A's own screen |
| 5 | Watch B's screen | The message arrives **within ~2 seconds, without any refresh** |
| 6 | Reply from B | Arrives on A the same way |
| 7 | Exchange ~10 messages quickly, alternating | All arrive, in the correct order, with no duplicates and no losses on either side |
| 8 | Send a message with Vietnamese diacritics and an emoji | Renders correctly on both ends — no `????`, no mojibake |

**Backend verification**
```bash
curl -s "https://umatter-apcs.duckdns.org/api/v1/social/chats/channels/{channelId}/messages" \
  -H "Authorization: Bearer $TOKEN_A" | jq 'length, .[-3:]'
```
Every message displayed on either device is present exactly once.

**Pass criteria** — Messages travel both directions in near-real-time, exactly once, in order, with
correct Vietnamese rendering.

**If nothing arrives** — Work the layers in this order before filing anything: (1) is `STOMP
connected successfully` in the log? (2) does social-api report a non-zero WebSocket session count?
(3) is RabbitMQ reachable and is the virtual host `/`? A failure at layer 1 with a successful HTTP
upgrade is the historical signature of the vhost bug.

---

### `E2E-M10-06` — Message history and ordering
**Priority** P1 · **Type** Persistence · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | Leave the chat room and re-enter on A | The full conversation loads, oldest → newest, sides correct |
| 2 | Force-stop and relaunch A, reopen the room | Same history — served from the API, not local state |
| 3 | Compare A's and B's views of the same conversation | Identical content and order |
| 4 | Check timestamps | Local ICT, sensible ordering, no 7-hour offsets |
| 5 | Scroll up through a longer conversation | Older messages load (or are all present); no gaps |

**Pass criteria** — History is complete, ordered, and identical on both devices after a cold restart.

---

### `E2E-M10-07` — Channel list, previews and unread counts
**Priority** P1 · **Type** Happy path · **Est.** 5 min

| # | Action | Expected result |
|---|---|---|
| 1 | With the chat closed on B, have A send 3 messages | — |
| 2 | On B, open the messages tab (friends) | The channel shows an **unread count of 3** and the latest message as its preview |
| 3 | Open the channel on B | Messages shown; the unread count clears |
| 4 | Go back to the list | Count is 0; the preview is the latest message |
| 5 | Have A send another while B sits on the **list** screen | Record whether the list updates live or on refresh — either is acceptable if consistent |
| 6 | Check the ordering of multiple channels | Most recently active first |

**Pass criteria** — Unread counts increment and clear correctly, and previews reflect the latest
message.

---

### `E2E-M10-08` — Friend profile
**Priority** P2 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | From the chat or friends list, open TEEN-B's profile | Name and avatar are shown |
| 2 | Observe what data is visible | **Without a grant**, no tracking data of TEEN-B is shown — only their basic identity |
| 3 | Confirm the sharing controls are present | The screen offers the category-based sharing options tested in [M13](E2E-M13-Privacy-Data-Access-Grants.md) |
| 4 | Navigate back | Returns cleanly to the chat/list |

**Pass criteria** — A friend's profile exposes identity only, until a grant exists.

**Note** — Step 2 is a privacy check. If TEEN-B's mood or sleep is visible to TEEN-A with no grant in
place, that is **S1**, and it contradicts showcase §10 directly.

---

### `E2E-M10-09` — Reconnection after a network interruption
**Priority** P1 · **Type** Recovery · **Est.** 6 min

The STOMP client is configured with a 5-second reconnect delay.

| # | Action | Expected result |
|---|---|---|
| 1 | With both devices in the chat, put **B** in aeroplane mode | B shows a disconnected state or simply stops receiving |
| 2 | Send 2 messages from A while B is offline | A's own view shows them as sent |
| 3 | Restore network on B | Within ~5–10 s the client reconnects — confirm the reconnect in logcat |
| 4 | Check B's screen | The 2 missed messages are present (delivered on reconnect or fetched on resume) — **no permanent gap** |
| 5 | Send from B immediately after reconnect | Delivers to A normally |
| 6 | Repeat with a Wi-Fi → mobile-data switch instead of aeroplane mode | Same recovery |

**Pass criteria** — The client reconnects automatically and no message is permanently lost.

**Note** — No push notification is expected for the offline period; see the warning above. What is
being tested here is **recovery**, not notification.

---

### `E2E-M10-10` — Unfriend
**Priority** P2 · **Type** Happy path · **Est.** 4 min

| # | Action | Expected result |
|---|---|---|
| 1 | Unfriend TEEN-B from TEEN-A | A confirmation is required |
| 2 | Confirm | B disappears from A's friends list |
| 3 | Check TEEN-B's list | A is gone there too |
| 4 | Check the chat channel | Record what happens to the direct channel and its history on both sides |
| 5 | Send a new friend request A→B afterwards | Allowed; accepting it restores a working channel |

**Pass criteria** — Unfriending is confirmed, mutual, and re-friending works afterwards.

---

## Exit criteria for M10

- **`M10-05` passes** — this is the case that proves real-time chat works at all.
- `M10-01`, `M10-03` pass — the friendship lifecycle works.
- `M10-06` passes — history is complete and consistent.
- `M10-09` passes — a network blip does not lose messages.
- `M10-08` step 2 passes — no tracking data leaks without a grant.

## Known issues / expected behaviour

| Observation | Status |
|---|---|
| No push notification when messaging an offline friend | ⚠️ **Known** — `message_sent` publish is commented out and the `message.missed` consumer was deleted. Do not file |
| `/ws` bypasses the Nginx gateway (Caddy routes it directly) | ✅ By design — REST health does not imply chat health |
| STOMP relay virtual host is pinned to `/` | ✅ Fixed 2026-07-20 — this is what makes chat work at all |
| Social publishes `social.*` domain events nobody consumes | ✅ Known dormant plumbing; invisible from the app |

## Run log

| Date | Build / commit | Devices + OS | Cases run | Pass | Fail | Blocked | Tester |
|---|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — | — |
