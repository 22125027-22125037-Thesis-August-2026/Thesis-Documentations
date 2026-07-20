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
  `sub` = the **profile id** (since the V6 users→profiles merge, deployed 2026-07-17, the profile id
  *is* the single identity — the old `users` table is gone), a redundant `profileId` claim kept for
  rollout compatibility, and `role`.
- **Access-token lifetime:** 1 hour (`MHSA_APP_JWTEXPIRATIONMS = 3600000`).

### Key distribution — JWKS (Dashboard) and static public key (everyone else)

Auth publishes the public half of its signing key as a JWKS document, and Dashboard consumes it.
The remaining services are still configured with the key directly via `JWT_PUBLIC_KEY` →
`mhsa.app.jwtPublicKey`. Both paths verify locally; neither calls back to Auth per request.

```
Auth (holds RSA private key) ──signs──► JWT  (kid in header)
   │
   ├─ GET /internal/v1/.well-known/jwks.json ──► Dashboard
   │     fetched lazily, cached by kid, refreshed when an unknown kid appears
   │     Dashboard is handed no signing material at all
   │
   └─ JWT_PUBLIC_KEY env (out of band, from the .env files)
         ──► Therapist, Tracking, AI, Social, Notification
             each parses it once at startup and verifies RS256 locally
```

**The endpoint** is `JwksController` in `auth-service`, returning `jwtUtils.getJwksResponse()`. It
sits under `/internal/**`, which `SecurityConfig` permits and the gateway does not route: a key set
is public information, but only services on the compose network need it.

**The consumer** is `JwksKeyProvider` in `shared-jwt`, active in any service that sets
`mhsa.app.jwksEndpoint`. Two properties worth stating because they are the parts that usually go
wrong:

- **The fetch is lazy, not at startup.** Fetching in `@PostConstruct` would make Auth a hard boot
  dependency for every consumer. Instead the first RS256 token triggers the fetch, so a consumer
  starts fine while Auth is still coming up — it just cannot validate until the first fetch lands.
- **Refresh is rate-limited to once a minute.** An unknown `kid` triggers a refetch, since that is
  what a rotation looks like from the consumer's side. Without a floor on the interval, tokens
  carrying forged kids would turn into a request amplifier aimed at Auth.

If a refresh fails, previously cached keys keep being served — a transient Auth outage does not
invalidate keys that are still good.

### What this buys, and the rotation question

Rotating the key pair no longer requires redeploying Dashboard. The more interesting property is
that **rotation does not have to log anyone out**, which is easy to assume it must.

Set `mhsa.app.jwtPreviousPublicKey` + `mhsa.app.jwtPreviousSigningKid` alongside the new pair and
Auth publishes *both* keys. New tokens are signed with the new kid; tokens already in the wild
carry the old kid and still find their key in the set. They expire naturally over the next hour and
clients pick up new ones. Drop the previous-key config once that window passes. That overlap is the
entire reason JWKS is a key *set* rather than a single key, and it is why rotation is a background
operation rather than a forced re-login for every user.

The five services still on `JWT_PUBLIC_KEY` do not get this — for them, rotation is still a
redeploy. Moving them over is a one-line compose change each (swap `MHSA_APP_JWTPUBLICKEY` for
`MHSA_APP_JWKSENDPOINT`); it is deliberately not done yet so that a JWKS or Auth-availability
problem can only affect one service.

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

This is how the dashboard summaries, AI's context fetch, and the reconciliation snapshots
(`/internal/grants`, `/internal/therapist-profiles`) stay private while still being callable between
services.

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

Because this is a mental-health product, another person (therapist, parent, or friend — grants are
profile-to-profile, so all three use the same mechanism) can **only** see a user's tracking data the
user has explicitly shared. This is the **data-access-grant** model:

```
Owner   → POST /api/v1/auth/grants { granteeProfileId, accessScope, expiresAt }   (consent)
        → recorded in auth_db.data_access_grants and replicated to tracking_db
          (auth.grant.created/revoked events + nightly reconcile — see 04-Event-Driven-Messaging §4)
Grantee → GET /api/v1/tracking/{moods|sleeps|foods|diaries|steps|breathing}/{ownerProfileId}
        → Tracking's AccessGuard allows: self, or an ACTIVE unexpired grant whose scope set
          covers THIS category. Otherwise 403.
```

**`accessScope` is a *set* of per-category tokens** — `READ_MOOD`, `READ_SLEEP`, `READ_FOOD`,
`READ_JOURNAL`, `READ_STEPS`, `READ_BREATHING`, or the `READ_ALL` shorthand — stored as a
comma-separated string in the single `access_scope` column (e.g. `READ_SLEEP,READ_FOOD`). One grant
per (granter, grantee) pair carries any subset, so a user can share sleep and nutrition but withhold
their journal. A grant also has an optional **`expiresAt`** (the mobile friend-profile screen issues
a 30-day expiry). Owners revoke at any time (`DELETE /api/v1/auth/grants/{granteeProfileId}`); grant
status is queryable (`GET /api/v1/auth/grants/status/{otherProfileId}`).

**Enforcement is per category** (`AccessScopes.allows(scopeCsv, token)` in `shared-contracts`, checked
by Tracking's `AccessGuard.canReadCategory(auth, profileId, 'READ_X')` on every per-category read).
`READ_ALL` covers everything; otherwise only the listed categories are readable. The mobile app mirrors
this: `DailyLogsSection` renders only the cards a grant covers, and the friend-profile grant sheet is a
category multi-select.

**The AI companion is a grantee like any other.** It is represented by a reserved system profile
(`SystemProfiles.AI_COMPANION_ID`, seeded in `auth_db`). Its grounding fetch
(`GET /internal/v1/tracking/context/{profileId}`) is gated in `ContextController`: Tracking returns
context only when the user holds an active grant to the AI principal, else **403** and the AI service
degrades to an ungrounded reply. Consent is **off by default** (default-deny), toggled from the chat
screen, and revocable — satisfying FR4/NFR1 (grounded *iff* granted). *Implemented 2026-07-20;
deploy pending (auth `V7`/`V8`, tracking `V9` migrations widen `access_scope` and seed the AI profile).*

---

## 6. Transport security (HTTPS — implemented July 2026)

- **Current:** production traffic is **HTTPS end-to-end at the edge**. A **Caddy** reverse proxy on
  the VM terminates TLS on `:443` for **`https://umatter-apcs.duckdns.org`** (Let's Encrypt,
  auto-renewed) and proxies `/api/*` etc. to the Nginx gateway on `:8080`. Release builds of the
  mobile app are HTTPS/WSS-only with cleartext forbidden, and the therapist web UI is served by
  Caddy at the domain root — the **same origin as the API**, so its requests need no CORS at all.
- The **mobile app is HTTPS/WSS in every build**, not just release: `axiosClient.ts` sets
  `BASE_URL = __DEV__ ? 'https://umatter-apcs.duckdns.org' : 'https://umatter-apcs.duckdns.org'`
  (both ternary branches are the same domain — the branch is vestigial), and `ChatScreen` defaults to
  `wss://umatter-apcs.duckdns.org/ws`. No raw IP appears anywhere in `thesis-mobile/src`.
- Raw-IP HTTP (`http://85.211.241.204:8080`) survives only in the **therapist web UI's
  `.env.development`** (a laptop `npm run dev` pointing at the VM, which is genuinely cross-origin)
  and in on-VM diagnostics. No shipping client uses it.
- Full detail (DuckDNS, Caddyfile routes, certificate handling, the migration order):
  [05-Deployment/04-DNS-HTTPS-and-Play-Release](../05-Deployment/04-DNS-HTTPS-and-Play-Release.md).

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
