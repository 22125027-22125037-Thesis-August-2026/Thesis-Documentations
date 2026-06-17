# Therapist Web UI — `therapist-web-ui`

| | |
|---|---|
| **Repo** | `therapist-web-ui` (public GitHub repo) |
| **Platform** | React **18**, Vite **5**, TypeScript, Tailwind CSS, Radix UI |
| **Audience** | Therapists |
| **Runs as** | a **Vite dev server on `:5173`** (the only non-Docker production piece) |
| **Backend** | the gateway, via `VITE_API_URL` (hard-codes the VM IP in `.env`) |

---

## 1. Purpose

The web UI is the **therapist's workspace**. Therapists manage their availability, see their assigned
patients (and a consenting patient's shared tracking summary), run video consultations, write clinical
notes, message patients, and review their dashboard.

It deliberately runs as a lightweight Vite dev server rather than a container — see the deployment
note below.

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
| Appointments | Therapist API bookings; **video join** returns Zoom credentials |
| Patients | Auth (`/api/v1/patients/{id}`) + Therapist API patients; **shared tracking context** via Tracking (gated by data-access grants) |
| Clinical notes | Therapist API `/api/v1/notes` |
| Messages | Social service (STOMP) |

`PermissionBadge` / `LockedCard` reflect the **consent/grant** and license-verification states — a
therapist only sees patient data they've been granted, and some actions unlock only after the
therapist's license is verified by an admin.

---

## 5. Configuration

| File | Holds |
|---|---|
| `.env`, `.env.development` | `VITE_API_URL` = the gateway (`http://<PUBLIC_IP>:8080`) — **hard-codes the VM IP** |
| `.env.example` | template |

> ⚠️ Because the IP is hard-coded in all three env files, a VM IP change requires updating them (the
> runbook does this with `sed`) and **restarting the Vite session**.

---

## 6. Deployment note (why it's not Dockerised)

On the OCI VM, the web UI runs as a **Vite dev server inside a tmux session** on `:5173`:
```bash
cd ~/therapist-web-ui && npm ci
tmux new-session -d -s web "cd ~/therapist-web-ui && npm run dev -- --host 0.0.0.0"
```
- It survives logout but **does not auto-start on reboot** — re-run after a VM reboot.
- Needs the host firewall (iptables) **and** the OCI security list to allow `5173`.
- It is the **only** non-Docker production component. `dist/` exists from `vite build` if you prefer
  to serve a static build instead.

In-repo docs: `therapist-web-ui/README.md`, `therapist-web-ui/docs/Therapist_Features.md`,
`Therapist_UI_Web.md`, `Manual_Test.md`, and the per-service controller references.
