# Therapist Web UI — `therapist-web-ui`

| | |
|---|---|
| **Repo** | [`therapist-web-ui`](https://github.com/22125027-22125037-Thesis-August-2026/therapist-web-ui) (public GitHub repo) |
| **Platform** | React **18**, Vite **5**, TypeScript, Tailwind CSS, Radix UI |
| **Audience** | Therapists |
| **Runs as** | a **static Vite build served by Caddy** from `/var/www/umatter-web` at the domain root (see §6). The old systemd Vite dev server on `:5173` is retired |
| **Backend** | the gateway — **same origin**, resolved from `window.location` in `src/lib/api/config.ts`; no host is baked into the bundle (see §5) |

---

## 1. Purpose

The web UI is the **therapist's workspace**. Therapists manage their availability, see their assigned
patients (and a consenting patient's shared tracking summary), run video consultations, write clinical
notes, message patients, and review their dashboard.

It deliberately ships as **static files served by Caddy** rather than as a container — there is no
Node process for it on the VM at all. See the deployment note below.

---

## 2. Tech stack

| Concern | Library |
|---|---|
| Framework / build | React 18 + Vite 5 + TypeScript |
| Styling | Tailwind CSS (+ `tailwindcss-animate`), `class-variance-authority`, `clsx`, `tailwind-merge` |
| UI primitives | **Radix UI** (dialog, dropdown, popover, select, tabs, toast, tooltip, avatar, switch, progress, scroll-area, separator, label, slot) |
| Routing | `react-router-dom` v6 |
| Charts | `recharts` (patient trend visualisations) |
| Real-time | `@stomp/stompjs` (chat with patients) |
| Dates | `date-fns` |
| Icons | `lucide-react` |

Scripts: `npm run dev` (Vite), `npm run build` (`tsc -b && vite build`), `npm run preview`,
`npm run lint`. Requires **Node 20**.

---

## 3. Source structure

```
therapist-web-ui/src/
├── main.tsx                # entry
├── App.tsx                 # app shell
├── router.tsx              # route definitions
├── pages/
│   ├── auth/               # login
│   ├── DashboardPage.tsx   # therapist dashboard summary
│   ├── appointments/       # upcoming/history, join video
│   ├── availability/       # slots + weekly templates
│   ├── patients/           # assigned patients, shared context
│   ├── clinical-notes/     # write/finalize notes
│   ├── messages/           # chat with patients
│   └── SettingsPage.tsx
├── components/
│   ├── layout/             # shell, nav
│   ├── ui/                 # Radix-based design system
│   ├── LockedCard.tsx      # gates features behind permissions/state
│   ├── PermissionBadge.tsx
│   └── StatusBadge.tsx
├── context/
│   ├── AuthContext.tsx     # JWT/session
│   └── I18nContext.tsx     # i18n
├── lib/api/                # typed API clients to the gateway
└── types/
```

---

## 4. Feature ↔ backend mapping

| Page | Talks to |
|---|---|
| Login / session | Auth service (`/api/v1/auth/login`, `/me`) |
| Dashboard | Therapist API (`/dashboard/summary`), Dashboard BFF |
| Availability | Therapist API availability + templates endpoints |
| Appointments | Therapist API bookings; **video join** returns a Jitsi room name, opened at `https://meet.jit.si/<room>` by `VideoSessionPage` |
| Patients | Auth (`/api/v1/patients/{id}`) + Therapist API patients; **shared tracking context** via Tracking (gated by data-access grants) |
| Clinical notes | Therapist API `/api/v1/notes` |
| Messages | Social service (STOMP) |

`PermissionBadge` / `LockedCard` reflect the **consent/grant** and license-verification states — a
therapist only sees patient data they've been granted, and some actions unlock only after the
therapist's license is verified by an admin.

**Appointment status badges.** The backend's three completion states (see
[Therapist-API §3](../02-Services/Therapist-API.md)) are deliberately *not* collapsed back into one
"Completed" chip — `StatusBadge` gives the therapist-actionable ones their own labels:
`PATIENT_COMPLETE` → **"Needs your note"**, `PROFESSIONAL_COMPLETE` → **"Awaiting patient review"**.
Filtering and "is this finished?" checks go through `isCompletedAppointmentStatus()` /
`COMPLETED_APPOINTMENT_STATUSES` in `src/lib/api/therapist.ts`, never a literal comparison, and
`AppointmentStatusServer` must mirror the backend enum **exactly** — a value the enum lacks (there is
no `NO_SHOW`) fails the entire multi-value `@RequestParam` conversion with a `500`.

---

## 5. Configuration

`src/lib/api/config.ts` defaults `API_BASE_URL` and `CHAT_WS_URL` to the **page's own origin**
(`window.location`), so **no host is baked into the production bundle** — the deployed UI is served
same-origin with the API and needs no CORS and no rebuild on an IP/domain change.

| File | Holds |
|---|---|
| `.env.development` | `VITE_API_BASE_URL=http://85.211.241.204:8080` + `VITE_CHAT_WS_URL=ws://85.211.241.204:8086/ws` — used only by `npm run dev` on a laptop, which *is* genuinely cross-origin and relies on the gateway's CORS `localhost` allow-list |
| `.env` | **Intentionally empty of variables** — comments only, explaining that the production build is same-origin so the API and chat URLs come from `window.location`, and that setting anything here would bake a host into the bundle and re-introduce cross-origin requests. It previously held five stale per-service vars (`VITE_AUTH_BASE_URL`, `VITE_THERAPIST_BASE_URL`, `VITE_SOCIAL_BASE_URL`, `VITE_NOTIFICATION_BASE_URL`, `VITE_TRACKING_BASE_URL`) from the pre-gateway design, read by no code and carrying **wrong ports**; those were removed on 2026-07-29 so laptop and VM match. **Never use this file as a port reference.** |
| `.env.example` | template |

> The old hard-coded-IP setup (every env file naming the VM, `sed` on migration) is gone — only the
> dev override still names an IP, and it affects nobody but a developer's laptop.

---

## 6. Deployment (static build behind Caddy)

On the Azure VM, the web UI is a **static Vite build served by Caddy** from `/var/www/umatter-web`
at the **domain root** — `https://umatter-apcs.duckdns.org`, the same origin as the API. Unknown
paths fall back to `index.html` (SPA routing); `/api/*`, `/ws*`, `/mhsa-media/*` are matched
explicitly by the Caddyfile so the SPA fallback can never swallow them.

Redeploying after a `git pull` on the VM:
```bash
cd ~/therapist-web-ui && npm ci && npm run build
sudo rsync -a --delete dist/ /var/www/umatter-web/
```
No Caddy reload needed — it serves the directory directly.

History: the UI previously ran as a systemd-managed Vite **dev server** on `:5173`
(`umatter-web.service`, before that a `tmux` session), addressing the VM by raw IP — the one client
the DuckDNS domain didn't cover. That service is **disabled/retired**; the NSG no longer needs
`5173`. See [05-Deployment/04-DNS-HTTPS-and-Play-Release §5](../05-Deployment/04-DNS-HTTPS-and-Play-Release.md).

In-repo docs: `therapist-web-ui/README.md`, `therapist-web-ui/docs/Therapist_Features.md`,
`Therapist_UI_Web.md`, `Manual_Test.md`, and the per-service controller references.
