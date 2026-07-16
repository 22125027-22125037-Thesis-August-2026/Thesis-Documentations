# Social Service

| | |
|---|---|
| **Port** | 8086 (host == container) — also a **public ingress** port |
| **Repo** | [`thesis_social`](https://github.com/22125027-22125037-Thesis-August-2026/thesis_social) (standalone Gradle repo) |
| **Java package** | `com.thesis.social` |
| **Database** | social DB (PostgreSQL 16, host 5436) |
| **Tech** | Spring Boot 3.3.4, Java 17, Spring Web, Security, Data JPA, **WebSocket**, AMQP, Flyway |
| **Gateway prefix** | `/api/v1/social/` → service `/api/v1/` (REST). WebSocket/STOMP uses `:8086` directly |

---

## 1. Purpose

The Social service reduces isolation with a **peer layer**: a friend graph (requests, accept/reject,
block) and **real-time 1:1 / group chat** delivered over **STOMP-over-WebSocket**. When a recipient
is offline, it emits a `message.missed` event so the Notification service can push them.

---

## 2. Package structure (feature-sliced)

```
com.thesis.social
├── friend/      # controller, dto, entity, repository, service
├── chat/        # controller (REST + WebSocket), dto, entity, repository, service
├── profile/     # local mirror of minimal profile data
├── common/      # entity, exception, util, web
├── config/      # WebSocket/STOMP + RabbitMQ relay config
├── event/       # event publishing (message.missed)
└── security/    # JWT filter
```

## 3. Data model (social DB)

| Table | Purpose |
|---|---|
| friend requests / friendships | the friend graph and request lifecycle |
| `blocks` | blocked profiles |
| chat `channels` | 1:1 and group channels |
| chat `messages` | messages + read receipts |
| `profiles` | local mirror of the minimal profile fields Social needs |

---

## 4. API surface

### Friends — `/api/v1/friends` (`FriendController`)
| Method | Path | Purpose |
|---|---|---|
| POST | `/requests` | send a friend request |
| DELETE | `/requests/{requestId}` | cancel a request |
| POST | `/requests/{requestId}/accept` | accept |
| POST | `/requests/{requestId}/reject` | reject |
| GET | `/requests/incoming` · `/requests/outgoing` · `/requests` | list requests |
| DELETE | `/{profileId}` | remove a friend |
| POST/DELETE | `/blocks/{profileId}` | block / unblock |

### Chat — `/api/v1/chats` (`ChatController`, REST) + `ChatWebSocketController` (STOMP)
| Method | Path | Purpose |
|---|---|---|
| POST | `/channels` | create a channel |
| GET | `/channels` | list channels |
| GET | `/channels/{channelId}/messages` | message history |
| PATCH | `/channels/{channelId}/messages/{messageId}/read` | mark read |
| *(STOMP)* | WebSocket | live send/receive of messages |

---

## 5. Real-time architecture (STOMP + RabbitMQ relay)

This is why Social runs its **own RabbitMQ** with extra plugins:

```
Mobile / Web (STOMP.js) ──WebSocket──► Social :8086 (STOMP broker relay)
                                          │
                                          ▼
                         Social RabbitMQ (rabbitmq:3.13-management)
                         + rabbitmq_stomp        (61613)
                         + rabbitmq_web_stomp     (15674 → host 15675)
```

- The mobile app and web UI both use **`@stomp/stompjs`** to subscribe to channel topics and publish
  messages in real time.
- Social's broker enables `rabbitmq_stomp` (host `61613`) and `rabbitmq_web_stomp` (host `15675`)
  precisely to relay chat to WebSocket clients — Auth's broker doesn't need these.
- Because real-time chat needs a direct socket, **`:8086` is opened to the internet** (one of the
  four public ingress ports), in addition to the gateway.

---

## 6. Events
- **Produces:** `message.missed` → `social.exchange` → Notification → FCM push (inbox type `CHAT`).
- **Consumes:** `therapist.assignment.changed` ← `booking.exchange` (the **core-stack** broker, via a
  second AMQP connection — see [04-Event-Driven-Messaging §4](../01-Architecture/04-Event-Driven-Messaging.md)).
  On every newly **ACTIVE** therapist↔patient match, `TherapistRelationshipService` creates their
  **friendship + direct chat channel** (idempotent). `INACTIVE` events are ignored, so a re-matched
  patient keeps the chat with their previous therapist. Queue
  `social.therapist.assignment.changed` (+ DLX/DLQ), manual ack, single consumer.

## 7. Security
- Stateless JWT (RS256), principal `profileId`. Friend/chat actions are scoped to the authenticated
  user; you can only act on your own requests/channels.

## 8. Run it
```bash
cd thesis_social && docker compose up -d --build
docker logs social-api | grep "Started"
```
In-repo references: `thesis_social/docs/SOCIAL_API_CONTROLLER_REFERENCE.md`,
`thesis_social/docs/Gemini_Context.md`, `thesis_social/CONTEXT.md`.
