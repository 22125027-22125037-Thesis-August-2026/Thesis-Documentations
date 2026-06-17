# Vibe-Coding a New Feature with Claude

> **Audience:** you (or a teammate) building a new feature with an AI coding assistant (Claude Code).
> This page tells the assistant *and you* what context to load, what rules to respect, and a
> repeatable recipe so AI-generated changes fit uMatter's architecture instead of fighting it.

---

## 1. Give the assistant the right context first

Before asking for code, point the assistant at:
1. This documentation set — especially
   [01-Architecture/01-System-Architecture](../01-Architecture/01-System-Architecture.md),
   [02-Service-Catalog-and-Ports](../01-Architecture/02-Service-Catalog-and-Ports.md), and the
   relevant [service doc](../02-Services/README.md).
2. The target repo's in-repo `CONTEXT.md` / `CONTEXT/` folder (the most code-adjacent truth).
3. The target repo's `docs/` controller references.

Each backend repo also has a `.claude/` directory — use it for repo-specific assistant config.

---

## 2. The non-negotiable rules (paste these into your prompt)

> When generating code for uMatter, you MUST respect:
> 1. **Database-per-service.** Only touch the current service's DB. Need another domain's data? Call
>    its REST API or consume its RabbitMQ event. **Never** add a cross-service `@ManyToOne`/`@JoinColumn`
>    — cross-domain references are **scalar UUID** columns (`profile_id`, `account_id`).
> 2. **Gateway-only public access.** New public endpoints live under `/api/v1/<domain>/…` and are
>    reached through Nginx `:8080`. Service-to-service endpoints go under `/internal/…` (the gateway
>    returns 403 for those).
> 3. **Schema via Flyway only.** Add a new `V<next>__name.sql`; never edit an existing migration;
>    `ddl-auto` is `validate`, so the entity must match the migration exactly.
> 4. **Ports are literals in `docker-compose.yml`**, never `.env` keys, never `${X_PORT:-N}`.
>    Spring services bind host==container port.
> 5. **Events:** publish **after** the local transaction commits; include a unique `messageId` +
>    `occurredAt` + the recipient `profile_id`; never block on a consumer.
> 6. **Security:** every public endpoint is authenticated (JWT RS256); enforce roles with
>    `@PreAuthorize`; principal is the `profileId` claim. Mental-health data is **grant-gated**.
> 7. **Secrets never go in code or GitHub.** New secrets become source-of-truth files on the laptop.

---

## 3. Recipes

### Add a REST endpoint to an existing service
1. Add the method to the relevant `*Controller`, under the service's existing base path.
2. Put logic in `service/`, data access in `repository/`; DTOs in `dto/`.
3. Add `@PreAuthorize` matching the role rules.
4. If it returns another domain's data, call that domain's API — don't reach into its DB.
5. Update [04-API-Reference](../04-API-Reference/README.md).
6. Rebuild just that service: `docker compose up -d --build <service>`; verify via `:8080`.

### Add a database table/column
1. New Flyway migration `V<next>__describe_change.sql`.
2. Add/adjust the JPA entity to match **exactly** (`validate` will fail otherwise).
3. Repository + service + controller as needed.

### Add a new notification/event
1. **Producer:** publish to your domain's topic exchange with a new routing key (+ `messageId`,
   `occurredAt`, `profile_id`).
2. **Consumer (Notification):** declare queue + DLX/DLQ in `RabbitConfig`, add strings to
   `application.yml` (`notification.rabbit.*`), add a DTO implementing `NotificationEnvelope`, write
   an `@RabbitListener` following the dual-action shape (idempotency → DB inbox → dispatch), map it to
   an inbox `type` + channel.
3. Update [04-Event-Driven-Messaging](../01-Architecture/04-Event-Driven-Messaging.md).

### Add a screen to the mobile app
1. New screen under `src/screens/<feature>/`; register it in `src/navigation/`.
2. API calls go through the `src/api/` clients (they target the gateway `BASE_URL`).
3. Reuse `src/components`, `src/context`, `src/hooks`, theme, and i18n locales.

### Add a page to the therapist web UI
1. New page under `src/pages/<feature>/`; add a route in `src/router.tsx`.
2. Use the Radix-based `components/ui` design system and `lib/api` clients.
3. Gate on consent/license with `PermissionBadge` / `LockedCard` where relevant.

---

## 4. After the assistant writes the code — your checklist

- [ ] Did it cross a service boundary with a DB join or another service's DB? → reject, use API/event.
- [ ] New public endpoint authenticated + role-checked?
- [ ] Schema change is a **new** Flyway migration and the entity matches?
- [ ] Compose ports still literal, host==container?
- [ ] Did it invent a port or hard-code a secret? → fix.
- [ ] Docs updated (port map / API ref / event topology) if affected?
- [ ] Rebuilt the service and verified through the gateway?

---

## 5. Things that commonly go wrong with AI changes

| Trap | Correct approach |
|---|---|
| AI adds a JPA relationship across services | scalar UUID + API/event |
| AI edits an old Flyway migration | add a **new** migration |
| AI re-introduces `${PORT:-x}` in compose | use a literal |
| AI exposes an `/internal/…` route to the client | route public traffic via `/api/v1/...` through the gateway |
| AI hard-codes the VM IP in code | use the configured `BASE_URL`/`VITE_API_URL`/env |
| AI forgets idempotency/`messageId` on a new event | every event carries a unique `messageId` |
