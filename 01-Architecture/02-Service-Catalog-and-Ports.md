# Service Catalog & Port Map

> **This is the single source of truth for ports and containers.** Every `docker-compose.yml` must
> agree with this page. Values verified against the deployed compose files (June 2026).

---

## 1. The unified port map (`host : container`)

Clients talk **only** to the gateway: in production via the Caddy HTTPS edge
(`https://umatter-apcs.duckdns.org` → Nginx `:8080`), in dev builds directly as
`http://<PUBLIC_IP>:8080`. Everything else is direct/admin. As of the 2026-05-25 alignment, every Spring service binds the **same port
inside and outside** its container (host == container), so `curl :8084/…` behaves identically on the
VM host and inside the container.

| # | Service | App port | PostgreSQL | pgAdmin | RabbitMQ AMQP | RabbitMQ Mgmt | Redis | Notes |
|---|---|---|---|---|---|---|---|---|
| 0 | **API Gateway (Nginx)** | **8080:8080** | – | – | – | – | – | sole frontend entry point |
| 1 | **Auth Service** | 8081:8081 | 5431:5432 | **5051:80** | 5671:5672 | 15671:15672 | 6371:6379 | sole pgAdmin (manages every DB) |
| 2 | **Notification API** | 8082:8082 | 5432:5432 | – | – | – | 6372:6379 | reuses Auth's RabbitMQ via host gateway |
| 3 | **Dashboard Service** | 8083:8083 | – | – | – | – | – | BFF, stateless, no own infra |
| 4 | **Tracking Service** | 8084:8084 | 5434:5432 | – | – | – | – | shares Auth's Redis/RabbitMQ |
| 5 | **Therapist API** | 8085:8085 | 5435:5432 | – | – | – | – | reuses Auth's RabbitMQ |
| 6 | **Thesis Social** | 8086:8086 | 5436:5432 | – | 5676:5672 | 15676:15672 | – | + STOMP 61613:61613, Web-STOMP 15675:15674 |
| 7 | **AI Service** | 8087:8087 | 5437:5432 | – | – | – | – | shares Auth's Redis/RabbitMQ |

- **MinIO:** `9000:9000` (S3 API) and `9001:9001` (console).
- **Therapist Web UI:** `5173` (Vite dev server — **not** Docker; a host process).

### Public ingress actually open on the VM
Only **`22, 8080, 8086, 5173`** are exposed to the internet (verified 2026-06-07):
- `22` SSH · `8080` gateway (everything the mobile app needs) · `8086` Social direct/STOMP-WS ·
  `5173` therapist web UI. Optionally `5051` for pgAdmin (otherwise SSH-tunnel it).

---

## 2. Why infra container ports don't match the host port

Each upstream image hard-codes its listen port and its ecosystem assumes it. We keep the **container**
port canonical and differentiate instances by the **host** port:

| Image | Canonical container port | Why it can't move |
|---|---|---|
| `postgres:*` | 5432 | every client default + `pg_isready` + Flyway target it |
| `redis:*` | 6379 | Spring default + `redis-cli ping` health probe |
| `rabbitmq:*-management` | 5672 / 15672 (+ 61613 STOMP, 15674 Web-STOMP for Social) | AMQP client defaults + management plugin |
| `dpage/pgadmin4` | 80 | image's internal nginx binds 80 |
| MinIO | 9000 / 9001 | already aligned with host |

> Social's Web-STOMP is host **15675** (not 15676) because 15676 is taken by its management UI; it
> gets the next free slot. Postgres `5436:5432` and Web-STOMP `15675:15674` are the only host≠container
> slots in Social's row.

---

## 3. The four Docker Compose stacks

The 7 services are deployed as **4 stacks** (one `docker-compose.yml` per repo). Bring-up order
matters — notification first (cross-stack consumers attach after it), auth last (heaviest build).

| Stack (repo) | Containers it runs | Build order |
|---|---|---|
| [**notification_api**](https://github.com/22125027-22125037-Thesis-August-2026/notification_api) | notification-service, notification-postgres, umatter-redis | 1st |
| [**therapist-api**](https://github.com/22125027-22125037-Thesis-August-2026/therapist-api) | therapist-api, therapist-postgres | 2nd |
| [**thesis_social**](https://github.com/22125027-22125037-Thesis-August-2026/thesis_social) | social-api, social-postgres, social-rabbitmq | 3rd |
| [**uMatter-Backend_Auth_Tracking_AI**](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) | nginx, auth/ai/tracking/dashboard services, postgres ×3, pgadmin, redis, rabbitmq, minio, minio-init | 4th (4 Spring services) |

Expected after a healthy bring-up: **20 running containers** + `minio-init` (exits 0 by design —
it is a one-shot bucket initialiser and the only service with no restart policy).

> Every stack declares the **`umatter-shared`** network as `external: true`. Create it **once**
> before any `compose up`: `docker network create umatter-shared`.

---

## 4. Per-service container inventory

### Stack 4 — `uMatter-Backend_Auth_Tracking_AI` (the "core" stack)
| Container | Image / build | Host ports |
|---|---|---|
| `nginx` | nginx:1.25-alpine | 8080 |
| `auth-service` | built (Spring Boot 4.0) | 8081 |
| `ai-service` | built | 8087 |
| `tracking-service` | built | 8084 |
| `dashboard-service` | built | 8083 |
| `postgres-auth` | postgres:16-alpine (`auth_db`) | 5431 |
| `postgres-ai` | postgres:16-alpine (`ai_db`) | 5437 |
| `postgres-tracking` | postgres:16-alpine (`tracking_db`) | 5434 |
| `pgadmin` | dpage/pgadmin4 (manages **all** DBs) | 5051 |
| `redis` | redis:7-alpine | 6371 |
| `rabbitmq` | rabbitmq:3.12-management-alpine | 5671 / 15671 |
| `minio` | minio/minio | 9000 / 9001 |
| `minio-init` | minio/mc (creates `mhsa-media` bucket, exits 0) | – |

### Stack 1 — `notification_api`
| Container | Image | Host ports |
|---|---|---|
| `umatter-notification-service` | built (Spring Boot 3.3.5) | 8082 |
| `notification-postgres` | postgres:16-alpine (`notification`) | 5432 |
| `umatter-redis` | redis:7-alpine | 6372 |

### Stack 2 — `therapist-api`
| Container | Image | Host ports |
|---|---|---|
| `therapist-api` | built (Spring Boot 4.0.5) | 8085 |
| `therapist-postgres` | postgres:15-alpine | 5435 |

### Stack 3 — `thesis_social`
| Container | Image | Host ports |
|---|---|---|
| `social-api` | built (Spring Boot 3.3.4) | 8086 |
| `social-postgres` | postgres:16-alpine | 5436 |
| `social-rabbitmq` | rabbitmq:3.13-management (+ STOMP, Web-STOMP) | 5676 / 15676 / 61613 / 15675 |

---

## 5. Admin UIs & credentials (development defaults)

> ⚠️ These are **development defaults**. They are documented for local/dev use; production secrets
> live only in the `.env` files on the laptop source-of-truth and the VM.

| Tool | URL (local) | Default credentials |
|---|---|---|
| **pgAdmin** (manages all Postgres) | `http://localhost:5051` | `admin@example.com` / `admin` |
| **RabbitMQ Management** (core) | `http://localhost:15671` | `guest` / `guest` |
| **RabbitMQ Management** (social) | `http://localhost:15676` | (see `thesis_social/.env`) |
| **MinIO Console** | `http://localhost:9001` | `minioadmin` / `minioadmin` |

Note the 2026-05-25 **pgAdmin consolidation**: previously each stack ran its own pgAdmin; now
**Auth's pgAdmin on 5051 is the only one** and has every Postgres instance registered (cross-network
DBs reached via `host.docker.internal:<host_port>`).

---

## 6. Health endpoints

| Service | Health check |
|---|---|
| Gateway | `GET :8080/health` → `healthy` |
| Auth / AI / Tracking | `GET :<port>/actuator/health` (8081 / 8087 / 8084) |
| Dashboard | `GET :8083/api/v1/dashboard/health` |
| Notification | `GET :8082/actuator/health` |
| Therapist / Social | no `/actuator/health` exposed — confirm via `docker logs … | grep "Started …Application"` |

Via the gateway: `GET :8080/api/v1/dashboard/health` proves gateway → dashboard end-to-end.
