# Dashboard Service (BFF)

| | |
|---|---|
| **Port** | 8083 (host == container) |
| **Repo / module** | [`uMatter-Backend_Auth_Tracking_AI/dashboard-service`](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) (Maven monorepo) |
| **Java package** | `com.mhsa.backend.dashboard` |
| **Database** | **none** — stateless |
| **Tech** | Spring Boot 4.0.2, Java 17, Spring Security (local RS256 verification) |
| **Gateway prefix** | `/api/v1/dashboard/` |

---

## 1. Purpose

The Dashboard service is a **Backend-for-Frontend (BFF)**. It owns no data. Its job is to **fan out**
to several internal services in parallel, then **compose** their responses into a single, client-
friendly summary — so the mobile app and web UI make one call instead of five.

This is a deliberate architectural pattern: it keeps client code simple and avoids chatty mobile
networking, while preserving the database-per-service boundary (Dashboard reads via each service's
**internal API**, never its database).

---

## 2. Package structure

```
com.mhsa.backend.dashboard
├── client/      # HTTP clients to auth/tracking/ai internal endpoints
├── config/
├── controller/  # DashboardController
├── dto/         # the composed response DTOs
├── jwt/         # SecurityUtils — reads the authenticated profile id off the SecurityContext
├── security/
└── service/     # parallel aggregation orchestration
```

---

## 3. API surface — `/api/v1/dashboard` (`DashboardController`)

| Method | Path | Purpose |
|---|---|---|
| GET | `/summary` | aggregated summary for the authenticated user |
| GET | `/context/{profileId}` | composed context for a given profile |
| GET | `/health` | health check (also the gateway's dashboard liveness probe) |

---

## 4. How aggregation works

```
GET /api/v1/dashboard/summary
   ├─► Auth     (internal): /internal/v1/profile/{profileId}/summary
   ├─► Tracking (internal): /internal/v1/dashboard/{profileId}/summary
   └─► AI       (internal): /internal/v1/dashboard/{profileId}/chat-stats
        ↓ (calls run in parallel)
   compose → single DashboardSummary DTO → client
```

Because the calls are parallelised, the client's latency is roughly the **slowest** upstream call,
not their sum.

---

## 5. Security

- Verifies RS256 JWTs **locally, using the RSA public key injected as `MHSA_APP_JWTPUBLICKEY`**
  (`${JWT_PUBLIC_KEY}` in compose) — exactly like every other service. Its `SecurityConfig` builds the
  shared `new JwtAuthenticationFilter(jwtUtils, userDetailsService)`.
- ⚠️ `MHSA_APP_JWKSENDPOINT` is set in compose and `application-docker.properties`, but **no code
  reads it and the endpoint it names does not exist** — Dashboard does *not* fetch a JWKS key set.
  See [01-Architecture/05-Security §1](../01-Architecture/05-Security-and-Authentication.md).
- Holds no database; it is purely a composition layer.

---

## 6. Run it

```bash
cd uMatter-Backend_Auth_Tracking_AI && docker compose up -d --build dashboard-service
curl http://localhost:8083/api/v1/dashboard/health
# end-to-end through the gateway:
curl http://localhost:8080/api/v1/dashboard/health
```
Depends on Auth, Tracking, and AI being healthy (declared in compose `depends_on`).

---

## 7. Design note
A BFF is the right tool **only** for read-time composition. It must stay stateless and must never
become a place where business rules or data ownership leak in — those belong in the owning domain
service.
