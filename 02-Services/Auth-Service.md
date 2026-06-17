# Auth Service

| | |
|---|---|
| **Port** | 8081 (host == container) |
| **Repo / module** | `uMatter-Backend_Auth_Tracking_AI/auth-service` (Maven monorepo) |
| **Java package** | `com.mhsa.backend.auth` |
| **Database** | `auth_db` (PostgreSQL 16, host 5431) |
| **Tech** | Spring Boot 4.0.2, Java 17, Spring Security, Spring Data JPA, Flyway, Redis, RabbitMQ, MinIO/S3 |
| **Gateway prefix** | `/api/v1/auth/`, `/api/v1/patients/` |

---

## 1. Purpose

Auth is the **identity backbone** of uMatter. It owns user accounts and profiles, issues and signs
JWTs (and publishes the public key via JWKS so every other service can verify them), manages the
**data-access-grant** consent model, stores avatars in MinIO, and handles **therapist license
verification** (admin) and therapist directory data.

Everything else in the system trusts a token that Auth signed.

---

## 2. Package structure

```
com.mhsa.backend.auth
├── client/        # outbound calls (e.g. therapist-api for reconcile)
├── config/        # security, S3, Redis, RabbitMQ config
├── controller/    # REST controllers (see API surface)
├── dto/
├── messaging/     # RabbitMQ producers/consumers
├── model/         # JPA entities (users, profiles, grants)
├── repository/
├── security/      # JWT signing, JWKS, filters
└── service/
```

---

## 3. Data model (`auth_db`)

| Table | Purpose |
|---|---|
| `users` | account credentials |
| `profiles` | app identity: role (`TEEN`/`THERAPIST`), full name, school, avatar URL |
| `data_access_grants` | consent records: grantor profile → grantee profile |
| therapist-profile / license tables | therapist directory info + license verification state |

10 Flyway migrations manage this schema.

---

## 4. API surface

### Public — `/api/v1/auth` (`AuthController`)
| Method | Path | Purpose |
|---|---|---|
| POST | `/register` | register a new user (teen/therapist) → returns tokens |
| POST | `/login` | login → access + refresh tokens |
| POST | `/refresh` | exchange a refresh token for a new access token |
| GET | `/me` | current user's profile |
| PATCH | `/profile` | update profile fields |
| POST | `/profile/avatar` | upload avatar (multipart → MinIO) |
| POST | `/password/change` | change password |
| GET | `/license` | therapist: read own license status |
| POST | `/license/renew` | therapist: submit/renew license |
| POST | `/logout` | logout |

### Data-access grants — `/api/v1/auth/grants` (`DataAccessGrantController`)
| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v1/auth/grants` | grant access to another profile |
| DELETE | `/api/v1/auth/grants/{granteeProfileId}` | revoke a grant |
| GET | `/api/v1/auth/grants/{profileId}` | grants this profile has given |
| GET | `/api/v1/auth/grants/{profileId}/received` | grants this profile has received |
| GET | `/api/v1/auth/grants/status/{otherProfileId}` | relationship status with another profile |

### Patients — `/api/v1/patients` (`PatientController`)
| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/patients/{profileId}` | patient profile lookup (used by therapist flows) |

### Admin — `/admin/v1/therapists` (`AdminTherapistController`)
| Method | Path | Purpose |
|---|---|---|
| POST | `/{id}/license/verify` | approve a therapist's license |
| POST | `/{id}/license/reject` | reject a therapist's license |

### Internal (service-to-service, gateway-blocked)
- `/internal/v1/.well-known/jwks.json` — **JWKS** public key set
- `/internal/v1/profile/{profileId}/summary` — profile summary for Dashboard
- `/internal/grants/...` — grant checks for other services
- `/internal/therapist-profiles/...` — therapist profile sync

---

## 5. Integrations

- **JWT/JWKS:** signs RS256 tokens; publishes the public key at the JWKS endpoint. The whole system
  depends on this. See [01-Architecture/05-Security-and-Authentication](../01-Architecture/05-Security-and-Authentication.md).
- **MinIO:** avatar uploads to bucket `mhsa-media`.
- **Therapist API:** `THERAPIST_API_BASE_URL = http://therapist-api:8085` and the shared
  `booking.exchange` are configured so Auth can do a **nightly assignment reconcile** and react to
  assignment events.
- **Redis / RabbitMQ:** shared infra for caching, refresh tokens, and messaging.

---

## 6. Security

- Issues tokens with `iss = mhsa.backend`, `aud = mhsa-api`, `profileId`, `role`; 1-hour access TTL.
- Implements the **Redis-backed refresh-token** half of the hybrid model.
- Admin endpoints require `ROLE_ADMIN`; profile/grant endpoints are self-scoped.

---

## 7. Run it

Part of the core stack — comes up with the monorepo's `docker-compose.yml`:
```bash
cd uMatter-Backend_Auth_Tracking_AI && docker compose up -d --build auth-service
# health:
curl http://localhost:8081/actuator/health
```
Key env (`.env` + compose): `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`, S3/MinIO keys, datasource,
Redis/RabbitMQ hosts. See [05-Deployment](../05-Deployment/03-Local-Development-Setup.md).

---

## 8. Related docs
- API details: [04-API-Reference](../04-API-Reference/README.md)
- Consent model: [01-Architecture/03-Data-Architecture §6](../01-Architecture/03-Data-Architecture.md)
- Original controller reference (verbatim, in-repo): `therapist-web-ui/docs/Authentication Service API controller.md`
