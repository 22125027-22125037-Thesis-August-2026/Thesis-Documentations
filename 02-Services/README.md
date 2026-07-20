# Backend Services

Deep-dive documentation for each of uMatter's seven backend services. Each service doc follows the
same template: **purpose → tech & location → data model → API surface → integrations → security →
how to run → known gaps**.

> Four services (Auth, AI, Tracking, Dashboard) live in **one Maven monorepo**
> (`uMatter-Backend_Auth_Tracking_AI`). Therapist, Social, and Notification are **separate Gradle
> repos**. See [01-Architecture/02-Service-Catalog-and-Ports](../01-Architecture/02-Service-Catalog-and-Ports.md).

---

## Responsibility matrix

| Service | Port | Repo | DB | One-line role |
|---|---|---|---|---|
| [Auth](Auth-Service.md) | 8081 | [uMatter-Backend](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) (monorepo) | `auth_db` | Identity, RS256 JWT issuance, profiles, grants, license verification |
| [AI](AI-Service.md) | 8087 | [uMatter-Backend](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) (monorepo) | `ai_db` | Gemini-powered companion grounded in user context |
| [Tracking](Tracking-Service.md) | 8084 | [uMatter-Backend](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) (monorepo) | `tracking_db` | Mood/sleep/food/diary/steps/breathing/streaks/treasures |
| [Dashboard](Dashboard-Service.md) | 8083 | [uMatter-Backend](https://github.com/22125027-22125037-Thesis-August-2026/uMatter-Backend_Auth_Tracking_AI) (monorepo) | none (BFF) | Aggregates other services for the clients |
| [Therapist API](Therapist-API.md) | 8085 | [therapist-api](https://github.com/22125027-22125037-Thesis-August-2026/therapist-api) | booking DB | Matching, availability, booking, Zoom, notes, reviews |
| [Social](Social-API.md) | 8086 | [thesis_social](https://github.com/22125027-22125037-Thesis-August-2026/thesis_social) | social DB | Friends + real-time STOMP chat |
| [Notification](Notification-API.md) | 8082 | [notification-api](https://github.com/22125027-22125037-Thesis-August-2026/notification_api) | `notification` | Event-driven push + email + inbox |

---

## What's shared across all services

- **Java 17**, **Spring Boot** (4.0.x or 3.3.x), **Spring Security** (stateless), **Spring Data JPA**.
- **Flyway** migrations with `ddl-auto: validate`.
- **RS256 JWT** verification (public key from `JWT_PUBLIC_KEY`; JWKS is not implemented).
- **Docker Compose** packaging; every stack joins the **`umatter-shared`** external network.
- **`/internal/...`** endpoints for service-to-service calls (blocked at the gateway).
- Configuration via **`.env`** (no port keys — ports are literals in compose).

## Reading order for a new developer

1. [Auth](Auth-Service.md) — start here; everything depends on identity & JWT.
2. [Tracking](Tracking-Service.md) — the richest data domain.
3. [Therapist API](Therapist-API.md) — the most complex business logic (booking/matching/video).
4. [AI](AI-Service.md), [Dashboard](Dashboard-Service.md), [Social](Social-API.md),
   [Notification](Notification-API.md).
