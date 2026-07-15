# Local Development Setup

> How to run the whole uMatter stack (or just the part you're working on) on a laptop. For the cloud
> deployment, see [02-Azure-Cloud-Runbook](02-Azure-Cloud-Runbook.md).

---

## 1. Prerequisites

| Tool | Version | For |
|---|---|---|
| **Docker Desktop** + Compose | latest | running every backend stack |
| **Java JDK** | 17 | building/running services outside Docker |
| **Node.js** | ≥ 20 | mobile app + web UI |
| **Git** | any | cloning |
| Android Studio / Xcode | – | mobile emulators (optional) |

The backend services build **inside Docker** (multi-stage Dockerfiles run Maven/Gradle), so you do
**not** need Maven/Gradle locally just to bring up the stack.

---

## 2. Get the repos

All six live under `D:\Y4-Sem 2 Thesis\`. If starting fresh, clone each from the
[`22125027-22125037-Thesis-August-2026`](https://github.com/22125027-22125037-Thesis-August-2026)
GitHub org (private repos need the deploy keys — see the runbook). The laptop copies already include
the gitignored `.env` files.

```bash
# Note: the notification repo is notification_api on GitHub but the compose/docs use notification-api locally
git clone https://github.com/22125027-22125037-Thesis-August-2026/notification_api.git notification-api
git clone https://github.com/22125027-22125037-Thesis-August-2026/therapist-api.git
git clone https://github.com/22125027-22125037-Thesis-August-2026/thesis_social.git
git clone https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI.git
git clone https://github.com/22125027-22125037-Thesis-August-2026/therapist-web-ui.git
git clone https://github.com/22125027-22125037-Thesis-August-2026/thesis-mobile.git
```

---

## 3. Create the shared network (once)

Every backend stack joins an **external** Docker network. Create it before any `compose up`:
```bash
docker network create umatter-shared
```

## 4. Bring up the backend (same order as production)

```bash
cd "D:/Y4-Sem 2 Thesis/notification-api"                  && docker compose up -d --build
cd "D:/Y4-Sem 2 Thesis/therapist-api"                     && docker compose up -d --build
cd "D:/Y4-Sem 2 Thesis/thesis_social"                     && docker compose up -d --build
cd "D:/Y4-Sem 2 Thesis/uMatter-Backend_Auth_Tracking_AI"  && docker compose up -d --build
```
Wait 2–3 minutes for health checks. Verify:
```bash
docker compose ps
curl http://localhost:8080/health                         # → healthy
curl http://localhost:8080/api/v1/dashboard/health        # → 200
```

> The mobile app/web UI expect the gateway at `:8080`. Locally that's `http://localhost:8080`.

---

## 5. Working on a single service (fast loop)

You can run one Spring service from your IDE against the Dockerised infra:
1. Start only the infra it needs (e.g. for Notification:
   `docker compose up rabbitmq redis notification-postgres`).
2. Export the env from that service's `.env`.
3. Run the `…Application` main class from the IDE. Flyway migrates the local DB on startup.

For the Gradle repos: `./gradlew bootRun` / `./gradlew bootJar`. For the Maven monorepo:
`./mvnw spring-boot:run -pl auth-service` (or build all with `./mvnw clean package`).

---

## 6. Run the frontends

### Mobile app
```bash
cd "D:/Y4-Sem 2 Thesis/thesis-mobile"
npm ci
npm run start          # Metro
npm run android        # build & launch
```
Set `BASE_URL` to a backend the device can reach. An Android emulator reaches the host via
`http://10.0.2.2:8080`; a physical device needs the laptop's LAN IP (and the gateway port open).

### Therapist web UI
```bash
cd "D:/Y4-Sem 2 Thesis/therapist-web-ui"
npm ci
npm run dev            # Vite on http://localhost:5173
```
Point `VITE_API_URL` (in `.env.development`) at `http://localhost:8080`.

---

## 7. Admin/inspection UIs (local)

| Tool | URL | Login |
|---|---|---|
| pgAdmin (all DBs) | http://localhost:5051 | `admin@example.com` / `admin` |
| RabbitMQ (core) | http://localhost:15671 | `guest` / `guest` |
| RabbitMQ (social) | http://localhost:15676 | see `thesis_social/.env` |
| MinIO console | http://localhost:9001 | `minioadmin` / `minioadmin` |

Inspect a DB directly:
```bash
docker exec -it postgres-auth psql -U postgres -d auth_db   # \dt, \l, SELECT …, \q
```

---

## 8. Common issues

| Symptom | Fix |
|---|---|
| `network umatter-shared not found` | `docker network create umatter-shared` (STEP 3) |
| Containers up but unhealthy | wait 30–60s (health checks are slow on first start); `docker logs <svc>` |
| Port already in use | find/kill the process, or stop a conflicting stack |
| DB connection errors | `docker compose down -v && up -d` to reset volumes (⚠️ wipes data) |
| Service ↔ service can't connect | confirm both stacks joined `umatter-shared` (`docker network inspect umatter-shared`) |
| Disk full | `docker system prune -a` |

---

## 9. Tear down
```bash
docker compose down            # stop & remove containers (keep volumes/data)
docker compose down -v         # ⚠️ also delete volumes (fresh DB next time)
```
