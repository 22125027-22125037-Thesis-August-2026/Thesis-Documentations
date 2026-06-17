# uMatter for Therapists — The Web Console 🩺

### Your remote practice, beautifully organized.

> **uMatter's Therapist Console is a complete web workspace for mental-health professionals.**
> Manage your availability, get matched with the patients who need you, run video and chat
> consultations, keep proper clinical records, and support young people between sessions — all from
> one clean, focused dashboard in your browser.

*This page is a product tour of what the console **does** for a therapist. For the engineering
behind it, see [03-Frontend/Therapist-Web-UI](../03-Frontend/Therapist-Web-UI.md).*

---

## ✅ 1. A trusted, verified profession — from day one

uMatter is built on trust, so onboarding takes credentials seriously.

- **Register** with your professional details: specialization, bio, years of experience, consultation
  fee, your **license number + issuing authority**, and **two required documents** (your license and a
  government ID).
- Your account then sits in a **review state** until an admin **verifies** your license — so every
  therapist patients meet on uMatter is a real, vetted professional. The license has a clear lifecycle
  (*Pending → Verified → Rejected / Expired*), and you renew it in a couple of clicks before it lapses.

*The result: patients trust the platform, and you practice alongside verified peers.*

---

## 📊 2. A dashboard that tells you what matters, now

The moment you log in, the **Dashboard** gives you the pulse of your practice:

- **Four KPIs** at a glance: patients you're following, sessions completed this month, your average
  rating, and draft notes waiting.
- **Today's schedule** — every session today, with a **"Join" button that lights up 10 minutes before
  start time**, and a Video/Text badge so you know the format.
- **This week** — sessions per day, so nothing sneaks up on you.
- **Pending actions** — booking requests awaiting your confirmation and draft notes to finalize.
- **Alerts** — patients **flagged by worrying mood data**, surfaced so you can reach out proactively.

---

## 📅 3. Appointments, fully under your control

A single **Appointments** hub organizes everything by **Requested / Upcoming / Today / Past /
Cancelled**, filterable by format and searchable by patient name. On each session you can:

- **Approve requests** — **Confirm** or **Reject** (with an optional reason that frees the slot). The
  decision window stays open while there's comfortable lead time before the session.
- **Join the session** — one click into a **secure video room**, or open the **chat** with the right
  patient.
- **Prepare in seconds** — see the patient's key info (age, contact, **emergency contact**) and their
  **5 most recent journal entries** *before* you meet, so you walk in with context.
- **Wrap up properly** — once a session is complete, write or open its **clinical note**.

---

## 🗓️ 4. Set your availability once — let it run

The **Availability** page makes scheduling effortless:

- A **weekly slot grid (8 AM–7 PM)**: click an empty cell to open a slot, click a slot to manage it.
  **Booked slots are read-only** and show the patient's name, so you never double-book.
- **Weekly templates** — define your recurring pattern (e.g. "Tuesdays 2–5 PM") and uMatter
  **automatically generates the next 30 days of slots** for you, topping them up week after week.
- Toggle a template on/off, or remove it — and uMatter cleans up the unbooked future slots while
  **protecting any session a patient has already booked**.

These open slots are exactly what your patients see when they book — your calendar and theirs stay
perfectly in sync.

---

## 👥 5. Know your patients — deeply, and with their consent

The **Patients** area is your clinical roster and patient record in one.

- **Roster:** every assigned patient with their status, assignment date, **risk-level badge**, and
  tags — filter and search to find anyone fast.
- **Risk level:** set each patient to **None / Low / Medium / High** so your highest-need patients
  stay visible.
- **Tags:** organize your caseload your way.
- A rich **patient profile** with tabs:
  - **Overview** — summary plus the patient's **matching-intake answers**, so you understand who
    they are and why they came.
  - **Clinical notes** — their full note history; start a new one in a click.
  - **Diary / Food / Sleep / Mood** — the patient's own self-tracking, with **charts** (7-day water
    intake, sleep duration & quality, a mood-trend line). **You see this only if the patient has
    granted access** — otherwise it stays respectfully locked.
  - **Sessions** — past and upcoming appointments, each linked to its clinical note.

> **Consent is built in.** Chat access and health-data access are *separate* permissions. A patient's
> journal and mood are private until they choose to share — and that boundary is enforced, not just
> promised.

---

## 💬 6. Stay connected between sessions

The **Messages** workspace is a real-time, three-pane chat console:

- **Live messaging** over WebSocket — messages send instantly, show read status, and display
  connection state.
- **Connection requests** — accept or decline patients who want to reach you.
- A **permission panel** beside each conversation shows whether the patient has shared their health
  data (with a jump to view it), and surfaces **mood alerts** when something needs attention.

---

## 📝 7. Clinical notes that respect your craft

The **Clinical Notes** editor is made for real clinical work, not an afterthought:

- Structured the way you already think — **SOAP** (Subjective · Objective · Assessment · Plan) — plus
  **Diagnosis**, **Recommendations**, and a **Summary**.
- **Risk flags** (suicidal ideation, self-harm, substance use, abuse): ticking one triggers a
  **"Safety protocol" prompt** — review the safety plan, check the emergency contact, schedule a
  prompt follow-up.
- **Save as draft** to keep refining, or **Finalize** to lock the record (and automatically mark the
  session complete).
- **Private by design:** *"Hồ sơ làm việc riêng tư — bệnh nhân không bao giờ thấy."* Your working
  notes are yours. Only the **Diagnosis + Recommendations** of a finalized note are shared back to the
  patient as their takeaways.

---

## 🎥 8. Run the session with everything at your fingertips

The **Video Session** view opens the meeting room and, beside it, a focused in-session sidebar:

- The patient's **reason for the visit** and their **5 most recent journal entries** for live context.
- A **scratchpad** that auto-saves your quick notes as you go.
- An **"End & write note"** button that carries your scratchpad straight into a new clinical note —
  pre-filling the *Subjective* section so writing up is fast while it's fresh.

---

## ⚙️ 9. A console that fits how you work

The **Settings** page keeps you in control:

- **Profile** — name, photo, specialization, experience, consultation fee, language, and bio.
- **Notifications** — stay informed of bookings, cancellations, messages, new data-access grants,
  mood alerts, and license-renewal reminders.
- **License** — view your status and **renew** with fresh documents in a guided flow.
- **Security** — change your password anytime.
- **Language** — work in **Tiếng Việt** or **English**.

---

## Why therapists choose the uMatter console

| You get… | Because… |
|---|---|
| 🧠 **Context before every session** | recent journal + matching intake + mood trends are right there |
| ⏱️ **Less admin** | recurring templates auto-generate your slots; scratchpad flows into notes |
| 🛡️ **Ethical guardrails** | consent-gated data, safety-protocol prompts, verified-only practice |
| 📑 **Real clinical records** | SOAP notes with draft/finalize and risk flags |
| 🔄 **Everything in sync** | your calendar, your patients, and their app are one connected system |

---

## In one sentence

**The uMatter Therapist Console gives you a verified, organized, consent-respecting workspace to
match with patients, run video & chat sessions with full context, and keep proper clinical records —
all from your browser.** 🩺
