# Security & Authentication

> How uMatter authenticates users, authorises actions, protects internal endpoints, and gates
> access to sensitive mental-health data.

---

## 1. Authentication model — stateless JWT (RS256)

uMatter uses **stateless** authentication. There is no server-side session; every request carries a
**JWT** in `Authorization: Bearer <token>`.

- **Issuer:** the **Auth service** mints tokens on register/login/refresh.
- **Algorithm:** **RS256** (asymmetric). Auth signs with a **private** RSA key; every other service
  verifies with the **public** key. No service needs the private key to validate a token.
- **Claims:** `iss = mhsa.backend`, `aud = mhsa-api`, a `kid` header for key identification,
  `profileId` (primary principal), `sub` (fallback principal), and `role`.
- **Access-token lifetime:** 1 hour (`MHSA_APP_JWTEXPIRATIONMS = 3600000`).

### Key distribution via JWKS
The Auth service publishes its public key at a **JWKS** endpoint:
```
http://auth-service:8081/internal/v1/.well-known/jwks.json
```
Services like Dashboard fetch the key from here (`MHSA_APP_JWKSENDPOINT`) and verify tokens locally —
no call to Auth per request. Other services (Therapist, Tracking, AI) are configured with the RSA
public key directly via `JWT_PUBLIC_KEY`. Both approaches verify the same RS256 signature.

```
Auth (holds RSA private key) ──signs──► JWT
       │ publishes public key
       ▼
JWKS endpoint  ◄──fetch──  Dashboard, …    every service verifies the signature locally,
JWT_PUBLIC_KEY env  ◄────  Therapist, Tracking, AI, Social, Notification
```

---

## 2. Authorisation — roles & RBAC

The JWT `role` claim is normalised to Spring authorities at each service:

| JWT `role` | Spring authority | Who |
|---|---|---|
| `TEEN` | `ROLE_PATIENT` | teenage end-users (mobile app) |
| `THERAPIST` | `ROLE_THERAPIST` | therapists (web dashboard) |
| `ADMIN` | `ROLE_ADMIN` | platform admins (license verification) |

Services use Spring Method Security (`@EnableMethodSecurity`) with `@PreAuthorize`. A common pattern
for profile-scoped reads (a user, or an admin, may read a given profile):
```java
@PreAuthorize("#profileId.toString() == authentication.name or hasRole('ROLE_ADMIN')")
```
The Therapist API also normalises roles so `hasRole('ROLE_X')` matches the literal authority
(`GrantedAuthorityDefaults("")`).

Examples of role rules in practice (Therapist API):
- **Booking / Reviews:** patient-only (`ROLE_PATIENT`), and a patient may only act on their own appointment.
- **Clinical notes:** therapist/admin-only, and only the *assigned* therapist may write the note.
- **Assignment lookup:** self-or-admin.

---

## 3. Protecting internal endpoints

Service-to-service calls use endpoints under **`/internal/...`**. These must never be reachable from
the public internet. Two layers enforce this:

1. **At the gateway:** Nginx returns **403** for any path starting with `/internal/`.
2. **At the network:** internal endpoints are called over the Docker network
   (e.g. `http://auth-service:8081/internal/...`), not the public IP.

This is how the JWKS endpoint, dashboard summaries, AI's context fetch, and grant checks stay private
while still being callable between services.

---

## 4. The gateway's security responsibilities (Nginx)

| Concern | How |
|---|---|
| **Single entry point** | only `:8080` is public; service ports aren't exposed via ingress |
| **CORS** | applied centrally; reflects allowed origins (VM IP any port, localhost/127.0.0.1 any port); upstream CORS headers stripped to avoid duplicates; `OPTIONS` short-circuited with 204 |
| **Internal blocking** | `location /internal/ { return 403; }` |
| **Upload limits** | `client_max_body_size 30m` for media |
| **Forwarded headers** | `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto` set on every proxy_pass |

---

## 5. Consent-gated data access (the privacy core)

Because this is a mental-health product, a therapist can **only** see a patient's tracking data the
patient has explicitly shared. This is the **data-access-grant** model:

```
Patient → POST /api/v1/auth/grants { granteeProfileId: <therapist profileId> }   (consent)
        → recorded in auth_db.data_access_grants and mirrored to tracking_db
Therapist → GET /api/v1/tracking/context/{patientId}   (Tracking checks the grant first)
        → no grant ⇒ 403 / empty; grant present ⇒ summarised context returned
```
The patient can revoke a grant (`DELETE /api/v1/auth/grants/{granteeProfileId}`). Grant status is
queryable (`GET /api/v1/auth/grants/status/{otherProfileId}`).

---

## 6. Transport security (current state & roadmap)

- **Current:** clients reach the gateway over **HTTP** at `http://<PUBLIC_IP>:8080`. The mobile app
  and web UI hard-code this IP. This is acceptable for the thesis demo but is the top hardening item.
- **Roadmap:** put a domain + TLS in front of the gateway (HTTPS), so clients stop hard-coding a raw
  IP over HTTP. When TLS is added, the **certificate + private key become new source-of-truth
  secrets** that must be copied back to the laptop per the 🔴 rule
  ([05-Deployment/02-Azure-Cloud-Runbook](../05-Deployment/02-Azure-Cloud-Runbook.md)).

---

## 7. Hybrid security design (intended architecture)

The system-wide intent is a **hybrid** model:
- **Stateless layer:** short-lived RS256 JWT access tokens on every API call (implemented everywhere).
- **Stateful layer:** long-lived **refresh tokens stored in Redis** for silent rotation and
  revocation (implemented in Auth; not yet wired into every downstream service — see the Therapist
  API "known gaps").

> ⚠️ **Access-token revocation (logout blacklist) is NOT effective today.** Auth's
> `TokenBlacklistService` *writes* a `blacklist:<token>` key to Redis on `POST /logout`, but the
> *enforcement* check in the shared `JwtAuthenticationFilter` is guarded by a `tokenBlacklistService`
> field that is **never set** (`setTokenBlacklistService(...)` is called nowhere; every service builds
> the filter with the 2-arg constructor that leaves it `null`). So no request path ever consults the
> blacklist — a logged-out **access token keeps working until it naturally expires (~1 hour)**. The
> blacklist is currently *write-only / inert* scaffolding. To activate it, wire the bean into the
> filter (e.g. `filter.setTokenBlacklistService(tokenBlacklistService)`) in each service's
> `SecurityConfig`. Logout *does* additionally call `refreshTokenService.revoke(...)` for the refresh
> token (enforcement of that path is separate and unverified here).

---

## 8. Secrets handling

- Secrets (`.env`, RSA keys, Firebase service-account JSON, Zoom credentials, Azure SSH key,
  GitHub deploy keys) are **never committed to GitHub**.
- The **laptop** `D:\Y4-Sem 2 Thesis` is the source of truth for all of them; the VM holds working
  copies. The full rule and rebuild procedure are in
  [05-Deployment/02-Azure-Cloud-Runbook](../05-Deployment/02-Azure-Cloud-Runbook.md).
- The JWT private key, Gemini API key, SMTP app password, and S3/MinIO keys all live in service
  `.env` files injected via Docker Compose `env_file`.
