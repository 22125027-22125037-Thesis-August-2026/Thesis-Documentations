# Thesis Context, Technology Justification & Future Work

> **Audience:** Professor & Council, and future contributors. Why uMatter is built the way it is, how
> its choices map to computer-science learning objectives, and where it goes next.

---

## 1. Thesis framing

**Title theme:** A microservices-based Mental Health Support Application for teenagers.
**Program:** Advanced Program in Computer Science (APCS), HCMUS — August 2026 cohort
(org `22125027-22125037-Thesis-August-2026`).

The thesis sets out to build — not just describe — a **production-grade distributed system** that
solves a real social problem (adolescent mental-health support) while demonstrating mastery of modern
software-engineering practice. The deliverable is a deployed, multi-client platform, documented end
to end (this folder).

### Demonstrated competencies
| Area | Evidence in the system |
|---|---|
| Distributed systems | 7 microservices, API gateway, service discovery via Docker DNS, cross-stack networking |
| Software architecture | DDD bounded contexts, BFF, event-driven design, clean layering |
| Databases | Database-per-service, schema versioning (Flyway), concurrency-safe booking |
| Security | RS256 JWT (asymmetric), RBAC, consent-gated data access |
| Cloud & DevOps | Containerisation, reproducible cloud deployment, documented DR runbook |
| AI integration | LLM (Gemini) grounded in user context |
| Full-stack | React Native mobile + React web, both on one gateway |
| Real-time systems | STOMP/WebSocket chat, FCM push, video consultations |

---

## 2. Why these technologies (justification)

| Decision | Rationale | Trade-off accepted |
|---|---|---|
| **Microservices over a monolith** | independent domains (auth, tracking, booking, social…) evolve and scale separately; clear ownership; teaches real distributed-systems concerns | operational complexity (mitigated by Docker Compose + a strict port map) |
| **Spring Boot (Java 17)** | mature ecosystem for secure REST + JPA + AMQP; strong typing; excellent for teaching layered architecture | heavier than Node for small services |
| **Database-per-service (PostgreSQL)** | enforces bounded contexts; no hidden coupling; each team owns its schema | cross-domain reads need APIs/events, not joins (by design) |
| **API Gateway (Nginx)** | one public surface; centralises CORS, routing, internal-endpoint blocking, upload limits | a single point to keep healthy (it's stateless and cheap) |
| **RabbitMQ events** | decoupled, resilient async comms; DLQ + retry + idempotency teach reliability engineering | eventual consistency (acceptable for notifications) |
| **JWT RS256** | stateless verification at every service without sharing the private key; industry-standard | key rotation is manual — the public key is an env var on every service, so rotating it means redeploying all of them. JWKS would make this tractable and is scaffolded (`JwtUtils.getJwksResponse()`) but **not wired up** — a concrete piece of future work. |
| **Redis** | fast cache + atomic idempotency primitive (`SETNX EX`) | another moving part |
| **MinIO (S3 API)** | keeps binaries out of Postgres; presigned URLs offload bandwidth; S3-compatible so cloud-portable | needs the gateway to proxy media |
| **Google Gemini 2.5 Flash** | strong, low-latency LLM; the AI is *grounded* in the user's own tracking data for personalisation | external dependency + cost |
| **React Native + React/Vite** | one stack family for two clients; large talent pool; fast iteration | RN native build complexity |
| **Zoom SDK (pluggable provider)** | reliable managed video; backend stays a thin authz gatekeeper; provider abstracted behind an interface (Jitsi alt implemented) | per-therapist Zoom accounts to dodge concurrency limits |
| **Single cloud VM, IaaS not PaaS** (Oracle free tier → **Azure**, 2026-07-11) | a real cloud VM at low cost; owning the orchestration is the point (a managed PaaS would have hidden it) | credit expiry → the 🔴 source-of-truth rule + runbook, which the Azure migration then **proved** by rebuilding the whole system from the laptop in one afternoon |
| **Domain + TLS edge (DuckDNS + Caddy)** rather than raw IPs | decouples clients from the host: the Oracle→Azure move needed **no mobile rebuild and no Play resubmission** | one more host service; the web UI still uses raw IPs and *did* need repointing |

---

## 3. Architectural highlights worth defending in a viva

1. **Strict bounded contexts.** No service reads another's database; cross-domain links are scalar
   UUIDs. This is the single most important property — it's what makes the services independently
   deployable. (See [01-Architecture/03-Data-Architecture](../01-Architecture/03-Data-Architecture.md).)
2. **Reliable eventing.** At-least-once RabbitMQ delivery is made safe with **Redis idempotency** +
   **DLQ/retry** so a user never gets a duplicate notification and a poison message never loops.
   (See [01-Architecture/04-Event-Driven-Messaging](../01-Architecture/04-Event-Driven-Messaging.md).)
3. **Consent-by-design privacy.** A therapist sees patient data **only** via an explicit, revocable
   data-access grant — appropriate for a mental-health product.
   (See [01-Architecture/05-Security-and-Authentication §5](../01-Architecture/05-Security-and-Authentication.md).)
4. **Concurrency-correct booking.** A single atomic conditional `UPDATE` prevents double-booking a
   slot without distributed locks. (See [Therapist-API §3](../02-Services/Therapist-API.md).)
5. **Grounded AI.** The companion's replies are personalised by injecting the user's own recent
   tracking context into the prompt — a concrete, defensible use of an LLM.
6. **Disaster recovery as documentation.** The system has no live backup; recoverability is achieved
   by a disciplined source-of-truth rule + a tested rebuild runbook — a pragmatic, honest engineering
   answer to a real constraint.

---

## 4. Known limitations (stated honestly)

| Limitation | Status |
|---|---|
| **HTTP, not HTTPS** | clients hard-code `http://<IP>:8080`; no TLS yet (top hardening item) |
| **IP hard-coding** | no domain/DNS; a VM rebuild requires repointing clients |
| **Booking policies pending** | 12h lead-time and 24h cancellation rules are specified but not yet enforced in code |
| **Refresh-token rotation partial** | implemented in Auth; not yet wired into every downstream service |
| **Test coverage** | meaningful integration tests exist (esp. Therapist API) but coverage is uneven |
| **No live data backup** | recovery is a fresh-seed rebuild; real data needs manual `pg_dump`/MinIO copy |
| **Single VM** | no horizontal scaling/HA yet (services are stateless and ready for it) |

---

## 5. Future work / roadmap

**Near-term hardening**
- Add a domain + **TLS/HTTPS** in front of the gateway; stop hard-coding a raw IP. (New cert/key
  become source-of-truth secrets per the 🔴 rule.)
- Enforce the **booking lead-time (12h)** and **cancellation (24h)** policies in code.
- Complete the **hybrid security** model (Redis refresh tokens everywhere; revocation).
- Broaden **automated test** coverage and add CI.

**Reliability & scale**
- **Kubernetes** deployment (manifests become source-of-truth files); horizontal scaling of stateless
  services; managed Postgres/Redis.
- A transactional **outbox** for event publishing (stronger producer guarantees than publish-after-commit).
- Centralised **observability** (structured logs, metrics, tracing across services).

**Product**
- AI **crisis-escalation** flow acting on the `ai.crisis.alerted` signal; clinician hand-off.
- Richer therapist analytics; group sessions; appointment reminders (`REMINDER`/`INSIGHT` inbox types).
- Parent role features (the seed data already models `PARENT` ↔ teen links).

---

## 6. Where the evidence lives

| Claim | Document |
|---|---|
| The whole architecture | [01-Architecture/01-System-Architecture](../01-Architecture/01-System-Architecture.md) |
| Every service in depth | [02-Services](../02-Services/README.md) |
| The deployed topology + DR | [05-Deployment](../05-Deployment/01-Deployment-Overview.md) |
| The API surface | [04-API-Reference](../04-API-Reference/README.md) |
| Reproducible test data | [06-Development/03-Testing-and-Accounts](../06-Development/03-Testing-and-Accounts.md) |
| Original, code-adjacent notes | each repo's `CONTEXT.md` / `docs/` folders |
