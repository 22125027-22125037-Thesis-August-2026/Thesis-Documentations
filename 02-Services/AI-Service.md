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
├── messaging/     # AuthEventListener (inert — see §7)
├── repository/
├── service/       # chat orchestration + Gemini call + context fetch
│                  #   + CrisisDetectionService + PiiScrubberService
└── util/          # AesEncryptor (JPA converter — chat content encrypted at rest)
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
   1. resolve profileId from JWT; persist the user message
   2. crisis gate (CrisisDetectionService): if the message matches a Vietnamese self-harm/violence
      keyword list (accent-insensitive), SKIP Gemini entirely and return a canned emergency reply
      with hotlines 115 / 19001567, flagged crisisDetected = true
   3. AI service → Tracking (internal): GET /internal/v1/tracking/context/{profileId}?days=7
      → Tracking checks for an active grant (user → AI-companion principal). No grant ⇒ 403,
        and the AI service returns a [USER CONTEXT - NOT SHARED] block (ungrounded reply). See §7.
   4. PII scrub (PiiScrubberService): emails, VN phone numbers, and VN personal names in the
      message, history, and context are replaced with placeholders before leaving the system
   5. build a prompt = system persona + scrubbed tracking context + last-10-message history + message
   6. AI service → Gemini 2.5 Flash: generateContent — Gemini must answer as JSON
      {reply, sentiment, is_crisis}, so the model also self-reports crisis
   7. persist the AI reply (chat content is AES-encrypted at rest via a JPA converter)
   8. return { reply, sentimentDetected, crisisDetected }
```

Configuration (`docker-compose.yml` env):
- `GEMINI_API_URL = https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- `GEMINI_API_KEY = ${GEMINI_API_KEY}` (secret, from `.env`)
- `SERVICE_TRACKING_URL = http://tracking-service:8084`
- `SERVICE_AUTH_URL = http://auth-service:8081`

---

## 6. Integrations

- **Auth service** — verification keys via JWKS (`MHSA_APP_JWKSENDPOINT`, `shared-jwt`'s
  `JwksKeyProvider`), resolved by `kid`. AI holds **no signing material**; as of 2026-07-20 it is no
  longer handed the RSA private key it never used.
- **Tracking service** (internal REST) — context grounding.
- **Google Gemini** (external REST) — the LLM.
- **Dashboard** consumes `/internal/v1/dashboard/{profileId}/chat-stats`.
- **Crisis signal:** the Therapist API's intake flow can emit `ai.crisis.alerted` on
  `booking.exchange` when a patient flags self-harm risk — a hook for AI/clinical escalation.

---

## 7. Safety, privacy & known gaps

**Implemented safeguards** (all in the send-message path, §5):
- **Crisis keyword guardrail** — `CrisisDetectionService` matches ~10 Vietnamese self-harm/violence
  phrases (diacritic-insensitive). On a hit, Gemini is never called; the user gets a fixed
  emergency-support reply pointing at hotlines **115** and **19001567**. There is **no** human
  escalation/alerting — deliberately (see the thesis's no-automatic-alerting rationale).
- **PII scrubbing** — `PiiScrubberService` masks emails, Vietnamese phone formats, and
  self-introduced/common Vietnamese full names before any text is sent to Google.
- **Encryption at rest** — `chat_messages.content` passes through `AesEncryptor`
  (a JPA `@Convert`er), so raw chat text is not readable in the DB.

**Consent-gated grounding** (implemented 2026-07-20): the AI companion is a **grantee** under the same
data-access-grant model as a therapist/parent/friend. It is a reserved system profile
(`SystemProfiles.AI_COMPANION_ID`, seeded in `auth_db`); Tracking's `ContextController` returns
grounding context **only** when the user holds an active grant to it, else **403** → the AI service
returns `[USER CONTEXT - NOT SHARED]` and replies without personal grounding. Consent is **off by
default**, toggled from the mobile chat screen (grant `READ_ALL`, no expiry), and revocable. This
satisfies FR4/NFR1 (grounded *iff* granted). *Deploy pending — needs auth `V8` (AI profile seed).*

**Known gaps** (verified in code):
- **`AuthEventListener` is inert scaffolding.** It declares plain queues (`auth.grant.created`,
  `auth.user.updated`, `auth.token.revoked`) that are **not bound to any exchange** (auth publishes
  grants on the `auth.events` topic exchange), and its handlers only log. The AI service therefore
  consumes **no** events today; the grant-replica consumer in Tracking is the real one.

---

## 8. Run it

```bash
cd uMatter-Backend_Auth_Tracking_AI && docker compose up -d --build ai-service
curl http://localhost:8087/actuator/health
```
Requires a valid `GEMINI_API_KEY` in `.env`. Depends on Auth + Tracking being healthy.

---

## 9. Notes & roadmap
- The persona/prompt template lives in the service layer; tune it there.
- Deeper safety/crisis handling (e.g. acting on `ai.crisis.alerted`, escalation flows) is an area
  for future hardening — see [07-Academic](../07-Academic/01-Thesis-Context-and-Future-Work.md).
