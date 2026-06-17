# Developer Onboarding

> **Audience:** a developer joining the uMatter team. This is your Day-1 → Week-1 guide. Read it
> top-to-bottom; it links out to everything else.

---

## Day 1 — Understand the system

1. Read [00-Overview/02-System-Overview](../00-Overview/02-System-Overview.md) (what the product does).
2. Read [01-Architecture/01-System-Architecture](../01-Architecture/01-System-Architecture.md) (how
   it's built) and skim [02-Service-Catalog-and-Ports](../01-Architecture/02-Service-Catalog-and-Ports.md)
   (the port map — keep this tab open forever).
3. Keep the [Glossary](../00-Overview/03-Glossary.md) handy for unfamiliar terms.

**The five mental models you must internalise:**
- **Gateway is the only public door.** Everything goes through Nginx `:8080`.
- **Database-per-service.** Never touch another service's DB; call its API or consume its event.
- **Cross-domain references are plain UUIDs**, never JPA joins.
- **Two integration channels only:** synchronous REST (incl. `/internal/…`) and async RabbitMQ events.
- **Secrets never hit GitHub.** The laptop `D:\Y4-Sem 2 Thesis` is the source of truth (🔴 rule).

---

## Day 2 — Run it locally

Follow [05-Deployment/03-Local-Development-Setup](../05-Deployment/03-Local-Development-Setup.md):
```bash
docker network create umatter-shared
# bring up the 4 stacks in order, then:
curl http://localhost:8080/health                  # → healthy
curl http://localhost:8080/api/v1/dashboard/health # → 200
```
Then run a frontend (web UI is the quickest: `cd therapist-web-ui && npm ci && npm run dev`).

Log in with a seeded test account (password `developer`) — see
[03-Testing-and-Accounts](03-Testing-and-Accounts.md).

---

## Day 3 — Find your way around a service

Every backend service follows the same Spring layout:
```
controller/   → REST endpoints (start here to find an API)
service/      → business logic
repository/   → JPA data access
entity|model/ → JPA entities (the DB schema in Java)
dto/          → request/response shapes
config/       → security, messaging, storage wiring
messaging|consumer/ → RabbitMQ producers/consumers
security/     → JWT filter
src/main/resources/db/migration/ → Flyway SQL (the real schema)
```
To understand a feature: trace `controller → service → repository`, and check `db/migration` for the
schema. Pick your service's deep-dive in [02-Services](../02-Services/README.md).

---

## Week 1 — Make a change

1. **Branch** off the repo's default branch (never commit straight to it).
2. **Schema change?** Add a new Flyway migration (`V<next>__describe.sql`) — never edit an old one,
   never rely on Hibernate auto-DDL (`ddl-auto: validate` will reject drift).
3. **New endpoint?** Add to the controller, respect the role rules
   ([05-Security](../01-Architecture/05-Security-and-Authentication.md)), and update
   [04-API-Reference](../04-API-Reference/README.md).
4. **New event?** Follow the recipe in
   [04-Event-Driven-Messaging §6](../01-Architecture/04-Event-Driven-Messaging.md).
5. **Test** locally (rebuild the one service: `docker compose up -d --build <service>`), then verify
   through the gateway.
6. **Update the docs here** that your change affects (port map / API ref / event topology).

---

## Conventions & invariants (don't break these)

| Rule | Why |
|---|---|
| Ports are **literals in `docker-compose.yml`**, never in `.env` | port topology belongs in committed config |
| No `${X_PORT:-N}` interpolation in compose | it's a known regression pattern |
| Spring services bind **host port == container port** | a `curl :8084` works the same inside & outside |
| `/internal/…` for service-to-service only | gateway blocks it from the internet |
| Producers publish **after** the local commit | never depend on a consumer being up |
| Keep the laptop copy of any new secret/config | the VM is disposable |

---

## Repository quick map

| Repo | What you'd change it for |
|---|---|
| `uMatter-Backend_Auth_Tracking_AI` | auth/identity, AI chat, tracking/self-care, the BFF, the **nginx gateway config** |
| `therapist-api` | booking, matching, availability, video, clinical notes, reviews |
| `thesis_social` | friends, real-time chat |
| `notification-api` | push/email/inbox, new notification events |
| `therapist-web-ui` | therapist web features |
| `thesis-mobile` | teen/patient mobile features |

---

## Who to ask / where to look
- In-repo `CONTEXT.md` / `CONTEXT/` folders hold the most detailed, code-adjacent notes.
- In-repo `docs/` folders hold controller references and manual test plans.
- This documentation set is the consolidated, cross-repo source of truth.
- Building a feature with an AI assistant? Read [02-Coding-with-Claude](02-Coding-with-Claude.md).
