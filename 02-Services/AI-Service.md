# AI Service

| | |
|---|---|
| **Port** | 8087 (host == container) |
| **Repo / module** | [`uMatter-Backend_Auth_Tracking_AI/ai-service`](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) (Maven monorepo) |
| **Java package** | `com.mhsa.backend.ai` |
| **Database** | `ai_db` (PostgreSQL 16, host 5437) |
| **Tech** | Spring Boot 4.0.2, Java 17, Spring Data JPA, Flyway, Redis, RabbitMQ |
| **External** | Google **Gemini 2.5 Flash** (`generativelanguage.googleapis.com`) |
| **Gateway prefix** | `/api/v1/ai/` |

---

## 1. Purpose

The AI service is uMatter's **mental-health companion**. A teen can chat with it any time. What makes
it more than a generic chatbot is **grounding**: before calling Gemini, the service pulls the teen's
own recent tracking context (moods, sleep, etc.) from the Tracking service, so replies are
**personalised** to how the user has actually been feeling.

---

## 2. Package structure

```
com.mhsa.backend.ai
├── config/        # Gemini client, security, JWT
├── controller/    # AiChatController (public), InternalController (dashboard)
├── dto/
├── entity/        # chat_sessions, chat_messages
├── exception/
├── messaging/     # event publishing
├── repository/
├── service/       # chat orchestration + Gemini call + context fetch
└── util/
```

## 3. Data model (`ai_db`)

| Table | Purpose |
|---|---|
| `chat_sessions` | a conversation thread per profile |
| `chat_messages` | the user/AI messages within a session |

---

## 4. API surface

### Public — `/api/v1/ai/chat` (`AiChatController`)
| Method | Path | Purpose |
|---|---|---|
| POST | `/send` | send a message → grounded Gemini reply (creates/continues a session) |
| GET | `/sessions` | list the user's chat sessions |
| GET | `/history/{sessionId}` | full message history for a session |

### Internal (gateway-blocked) — `/internal/v1/dashboard` (`InternalController`)
| Method | Path | Purpose |
|---|---|---|
| GET | `/{profileId}/chat-stats` | chat usage stats for the Dashboard BFF |

---

## 5. The grounded-chat flow

```
POST /api/v1/ai/chat/send  { message }
   1. resolve profileId from JWT
   2. AI service → Tracking (internal): GET /internal/v1/tracking/context/{profileId}
   3. build a prompt = system persona + user's tracking context + conversation history + message
   4. AI service → Gemini 2.5 Flash: generateContent
   5. persist user message + AI reply to chat_messages
   6. return the reply
```

Configuration (`docker-compose.yml` env):
- `GEMINI_API_URL = https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- `GEMINI_API_KEY = ${GEMINI_API_KEY}` (secret, from `.env`)
- `SERVICE_TRACKING_URL = http://tracking-service:8084`
- `SERVICE_AUTH_URL = http://auth-service:8081`

---

## 6. Integrations

- **Tracking service** (internal REST) — context grounding.
- **Google Gemini** (external REST) — the LLM.
- **Dashboard** consumes `/internal/v1/dashboard/{profileId}/chat-stats`.
- **Crisis signal:** the Therapist API's intake flow can emit `ai.crisis.alerted` on
  `booking.exchange` when a patient flags self-harm risk — a hook for AI/clinical escalation.

---

## 7. Run it

```bash
cd uMatter-Backend_Auth_Tracking_AI && docker compose up -d --build ai-service
curl http://localhost:8087/actuator/health
```
Requires a valid `GEMINI_API_KEY` in `.env`. Depends on Auth + Tracking being healthy.

---

## 8. Notes & roadmap
- The persona/prompt template lives in the service layer; tune it there.
- Safety/crisis handling (e.g. acting on `ai.crisis.alerted`, escalation flows) is an area for
  future hardening — see [07-Academic](../07-Academic/01-Thesis-Context-and-Future-Work.md).
