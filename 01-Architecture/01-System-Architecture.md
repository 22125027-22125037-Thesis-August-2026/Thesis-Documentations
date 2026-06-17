# System Architecture

> The engineering blueprint of uMatter: the patterns, the component topology, the request paths,
> and the design rules that hold the system together.

---

## 1. Architectural style

uMatter is a **microservices** system organised around **Domain-Driven Design** bounded contexts.
The guiding principles, and where each is realised:

| Principle | What it means here | Enforced by |
|---|---|---|
| **Microservices** | Independent, separately-deployable services | 5 repos, 7 services, 4 Docker stacks |
| **API Gateway** | One public entry point; clients never call services directly | Nginx `:8080` (`nginx/nginx.conf`) |
| **Database per Service** | Each service owns its schema; no shared tables | 7 isolated PostgreSQL instances |
| **No cross-DB coupling** | Cross-domain links are plain UUID fields, never JPA joins | `account_id`, `profile_id` scalar columns |
| **Backend for Frontend** | A service composes others for the client | Dashboard service |
| **Event-Driven** | Producers announce facts; consumers react asynchronously | RabbitMQ topic exchanges |
| **Stateless auth** | No server session; every request carries a JWT | RS256 JWT + JWKS |
| **Externalised config** | Ports/keys/secrets via env + compose, not hard-coded | `.env`, `docker-compose.yml` |

---

## 2. Component topology

```
                          INTERNET
                              │
   ┌──────────────────────────┼───────────────────────────┐
   │                          │                            │
┌──┴───────────┐      ┌───────┴────────┐          ┌────────┴─────────┐
│ Mobile app   │      │ Therapist Web  │          │ Zoom / Firebase  │
│ React Native │      │ React + Vite   │          │ Gemini (external)│
│ (teens)      │      │ :5173 (teens?) │          └──────────────────┘
└──────┬───────┘      └───────┬────────┘
       │  REST + WS           │ REST + WS
       └──────────┬───────────┘
                  ▼
        ╔══════════════════════╗
        ║  NGINX API GATEWAY   ║  :8080   ── /health, CORS, /internal/ → 403
        ╚══════════╤═══════════╝
   routing by path prefix (see §4)
   ┌─────────┬─────────┬─────────┬──────────┬──────────┬──────────┬───────────┐
   ▼         ▼         ▼         ▼          ▼          ▼          ▼           ▼
┌──────┐ ┌──────┐ ┌────────┐ ┌─────────┐ ┌─────────┐ ┌───────┐ ┌──────────────┐
│ Auth │ │  AI  │ │Tracking│ │Dashboard│ │Therapist│ │Social │ │ Notification │
│:8081 │ │:8087 │ │ :8084  │ │  :8083  │ │  :8085  │ │ :8086 │ │    :8082     │
└──┬───┘ └──┬───┘ └───┬────┘ └────┬────┘ └────┬────┘ └───┬───┘ └──────┬───────┘
   │        │         │           │           │          │            │
 auth_db   ai_db   tracking_db  (stateless) booking_db social_db  notification_db
   │        │         │                       │          │            │
   └────────┴─────────┴───────────┬───────────┴──────────┴────────────┘
                                  ▼
   Shared infra:  Redis  ·  RabbitMQ  ·  MinIO (S3)
   Cross-stack reachability: the `umatter-shared` external Docker network
```

> **Important port note:** the *internal* service ports above are the **real** ones from the
> deployed `docker-compose.yml` files (AI = **8087**, Tracking = **8084**, Dashboard = **8083**).
> An older `README.md` in the backend repo lists AI/Tracking/Dashboard as 8082/8083/8084 — that is
> **stale**. The canonical map is [02-Service-Catalog-and-Ports](02-Service-Catalog-and-Ports.md).

---

## 3. The two integration channels

Services integrate **only** through these two channels — never by sharing a database.

### 3.1 Synchronous REST (request/response)
Used when a caller needs data **now**.

- **Client → Gateway → Service:** all public traffic. Path-prefixed (`/api/v1/<domain>/`).
- **Service → Service (internal):** over the Docker network, using `/internal/...` endpoints.
  Examples:
  - Dashboard → Auth `/internal/v1/profile/{id}/summary`, → Tracking `/internal/v1/dashboard/{id}/summary`, → AI `/internal/v1/dashboard/{id}/chat-stats`
  - AI → Tracking `/internal/v1/tracking/context/{profileId}` (to ground the prompt)
  - Auth → Therapist `http://therapist-api:8085` (nightly assignment reconcile)
- **Cross-stack reachability** is provided by the **`umatter-shared`** external network, which
  every Compose stack joins. Without it, `compose up` fails ("network not found").

### 3.2 Asynchronous events (publish/subscribe)
Used when a producer wants to **announce a fact** and not wait. Carried by RabbitMQ **topic
exchanges**; consumers bind queues with routing keys. The Notification service is the principal
consumer. Full topology: [04-Event-Driven-Messaging](04-Event-Driven-Messaging.md).

---

## 4. Request routing (the gateway)

Nginx routes purely by **URL path prefix**. Public routes strip/forward to the right upstream;
`/internal/` is hard-blocked.

| Public path (client calls `:8080…`) | Upstream service:port | Notes |
|---|---|---|
| `/api/v1/auth/` | auth-service:8081 | |
| `/api/v1/patients/` | auth-service:8081 | |
| `/api/v1/ai/` | ai-service:8087 | |
| `/api/v1/tracking/` | tracking-service:8084 | |
| `/api/v1/dashboard/` | dashboard-service:8083 | BFF |
| `/api/v1/therapist/` | therapist-api:8085 | rewritten to the service's `/api/v1/` |
| `/api/v1/social/` | social-api:8086 | rewritten to the service's `/api/v1/` |
| `/api/v1/notification/` | notification-service:8082 | rewritten to the service's `/api/v1/` |
| `/mhsa-media/` | minio:9000 | presigned media GET (Host preserved for SigV4) |
| `/internal/` | — | **returns 403** (never exposed) |
| `/health` | — | returns `healthy` (gateway liveness) |

CORS is applied centrally at the gateway and **stripped from upstream** responses to avoid
duplicate headers. Allowed origins: the VM's public IP on any port, and `localhost`/`127.0.0.1`
on any port (covers the Vite dev server on `:5173`). Body size is raised to **30 MB** for media
uploads. See [05-Security-and-Authentication](05-Security-and-Authentication.md) and the gateway
section of [02-Service-Catalog-and-Ports](02-Service-Catalog-and-Ports.md).

---

## 5. Technology stack summary

| Layer | Technology |
|---|---|
| **Backend frameworks** | Spring Boot **4.0.2** (Auth/AI/Tracking/Dashboard), **4.0.5** (Therapist), **3.3.x** (Social, Notification); Java **17** |
| **Build** | Maven (backend monorepo), Gradle (Therapist, Social, Notification) |
| **Persistence** | PostgreSQL 15/16, Spring Data JPA + Hibernate, **Flyway** migrations (`ddl-auto: validate`) |
| **Messaging** | RabbitMQ (topic exchanges, DLQ), Spring AMQP |
| **Cache / dedupe** | Redis (Spring Data Redis) |
| **Object storage** | MinIO (S3 API, AWS SDK / presigned URLs) |
| **Auth** | JWT RS256, JWKS endpoint, Spring Security (stateless) |
| **Real-time** | STOMP over WebSocket (Social chat), WebSocket relay via RabbitMQ STOMP plugin |
| **Push / email** | Firebase Admin SDK (FCM), SMTP + Thymeleaf templates |
| **Video** | Zoom Meeting SDK (pluggable provider; Jitsi alternative implemented) |
| **AI** | Google Gemini 2.5 Flash (REST) |
| **Gateway** | Nginx 1.25 |
| **Mobile** | React Native 0.83.1, React 19, TypeScript, React Navigation, Axios, STOMP.js, Notifee + Firebase Messaging |
| **Web** | React 18, Vite 5, TypeScript, Tailwind, Radix UI, React Router, Recharts |
| **Containerisation** | Docker + Docker Compose |
| **Cloud** | Oracle Cloud Infrastructure (OCI) |

Justification for these choices is in [07-Academic](../07-Academic/01-Thesis-Context-and-Future-Work.md).

---

## 6. Cross-cutting design rules (the invariants)

These rules are *non-negotiable* and any new code or AI-generated change must respect them:

1. **No cross-service database access.** A service may only touch its own DB. Need another domain's
   data? Call its API or consume its event. Cross-domain IDs are stored as **plain UUID columns**,
   never `@ManyToOne`/`@JoinColumn`.
2. **The gateway is the only public door.** Clients call `:8080`. Direct service ports and all
   `/internal/` endpoints are not for public consumption.
3. **Producers don't depend on consumers.** Publish the event after the local transaction commits;
   never block on a downstream service being up.
4. **Ports live in `docker-compose.yml`, not `.env`.** Port values are literals in compose; `.env`
   holds **zero** port keys (a deliberate 2026-05 cleanup). Re-introducing `${X_PORT:-N}` is a
   regression.
5. **Schema changes go through Flyway.** Never rely on Hibernate auto-DDL; services run with
   `ddl-auto: validate`.
6. **Secrets never reach GitHub.** `.env`, keys, and credentials live only on the laptop
   source-of-truth and on the VM. See the 🔴 rule in
   [05-Deployment/02-Oracle-Cloud-Runbook](../05-Deployment/02-Oracle-Cloud-Runbook.md).

---

**Next:** [02-Service-Catalog-and-Ports](02-Service-Catalog-and-Ports.md) for the canonical port
map, or [03-Data-Architecture](03-Data-Architecture.md) for the database design.
