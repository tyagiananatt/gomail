# ReachInbox Production-Grade Email Scheduler & Dashboard 🚀

A production-grade, distributed email scheduler service and interactive management dashboard built with **TypeScript**, **Express.js**, **BullMQ**, **Redis**, **Prisma ORM**, **Ethereal Fake SMTP**, **Elasticsearch**, **React (Vite)**, and **Tailwind CSS**.

---

## 📸 Figma UI Match Overview

The frontend interface is a pixel-perfect replica of the provided ReachInbox Figma design:
1. **Google Login / Authentication** (`/login`): Centered card with Google 4-color sign-in, email/password fallback, and subtle mint green accents.
2. **Scheduled Emails View**: Left sidebar with `ONB` brand typography, user profile card (Oliver Brown), green outlined "Compose" pill, navigation badges (`Scheduled 12`), search input, and orange scheduled time pills (`⏰ Tue 9:15:12 AM`).
3. **Sent Emails View**: Navigation badge (`Sent 785`), gray sent pills (`Sent`), real-time snippet previews, and star toggles.
4. **Email Detail View**: Back navigation `←`, sender avatar initial, rich callout block with yellow accent and lightning badges (`⚡ Extremely Exclusive... ⚡`), attachment preview cards (`Tennis_Coach_Profile.png 1.2 MB`), and live Ethereal inspection URLs.
5. **Compose New Email Modal**: Multi-recipient tag input, "+N more" badges, **CSV Lead List Uploading**, sender dropdown, inter-email delay throttle, hourly limit fields, and rich formatting toolbar.

---

## ⚡ Architectural Highlights & Answers to Assignment Requirements

### 1️⃣ Core Scheduler (No Cron Jobs!)
- **Why No Cron?** Traditional schedulers using cron (`node-cron`, `agenda`) poll every minute, causing database read pressure, clock jitter (up to 59s lag), and distributed race conditions.
- **The BullMQ Solution**: BullMQ uses **Redis Sorted Sets (`ZSET`)** with sub-millisecond timestamps as scores. When a job matures, Redis triggers the event to exactly one worker.
- **Persistence & Restart Safety**:
  - Jobs are stored durably in Redis and DB.
  - On application boot, `reconcilePendingEmails()` checks SQLite/Postgres for any pending `SCHEDULED` emails and re-enqueues them to BullMQ with `delay = max(0, scheduledAt - now)`.
  - **Idempotency**: Every job uses `jobId = email.id`. The worker verifies that status is still `SCHEDULED` before sending. Duplicate dispatches are mathematically impossible.

### 2️⃣ Concurrency, Delay & Rate Limiting
- **Worker Concurrency**: Configured via `WORKER_CONCURRENCY=5` (adjustable in `.env`). Multiple jobs process concurrently across CPU cores without blocking.
- **Inter-Email Delay Throttle**: Minimum 2 seconds between individual email sends (configurable via `DEFAULT_DELAY_BETWEEN_EMAILS_SECONDS=2` or custom per batch). This mimics provider throttling and prevents ESP spam classification.
- **Hourly Rate Limiting (Per-Sender)**:
  - Atomic Redis counters keyed by `email_rate:{senderEmail}:{YYYY-MM-DD-HH}` with 2-hour TTL.
  - Multi-worker safe: uses atomic `INCR` operations.
  - **Next Hour Window Rescheduling**:
    - When `counter > hourlyLimit`, **jobs are NEVER dropped or failed**.
    - The worker calculates the milliseconds remaining until the top of the next hour window (`nextHour = setHours(h + 1, 0, 0, 0)`).
    - Reschedules the job in BullMQ with that delay, preserving queue order.
    - Updates database status and records the rate limit event.

### 3️⃣ Slack Alerts on Rate Limit Hit
- Live Slack integration supporting **Incoming Webhooks** and **OAuth**.
- The instant an hourly rate limit threshold is crossed for a sender, the worker dispatches a rich Slack Block Kit message with sender details and the next send window time.
- **Resilience**: If Slack is disconnected, the worker logs cleanly and continues without crashing. When connected, notifications resume immediately with no server restart.

### 4️⃣ Elasticsearch Full-Text Search with DB Fallback
- Scheduled and sent emails are indexed in Elasticsearch (`@elastic/elasticsearch`).
- Real-time search uses fuzzy multi-match across `subject^3`, `recipientEmail^2`, `recipientName^2`, and `body`.
- **Zero-Downtime Fallback**: If Elasticsearch is not running, searches automatically and transparently fall back to relational DB queries.

### 5️⃣ Live BullMQ Monitoring Dashboard
- Embedded Bull-Board (`@bull-board/express`) available live at:
  `http://localhost:5000/admin/queues`
- Monitor active, waiting, delayed, and completed jobs visually.

---

## 🛠️ Step-by-Step Interview Preparation Guide

We have created an in-depth, file-by-file documentation folder in `docs/` specifically for preparing your presentation:
- [01_PROJECT_ARCHITECTURE_AND_SETUP.md](docs/01_PROJECT_ARCHITECTURE_AND_SETUP.md) — High-level architecture & data flow diagrams.
- [02_DATABASE_AND_MODELS.md](docs/02_DATABASE_AND_MODELS.md) — Prisma schema, models, idempotency, and crash resilience.
- [03_BULLMQ_AND_REDIS_SCHEDULER.md](docs/03_BULLMQ_AND_REDIS_SCHEDULER.md) — Why no cron, Redis sorted sets, concurrency, and rate-limiting math.
- [04_EMAIL_SENDER_AND_ETHEREAL_SMTP.md](docs/04_EMAIL_SENDER_AND_ETHEREAL_SMTP.md) — Ethereal fake SMTP setup and preview URLs.
- [05_ELASTICSEARCH_INDEXING_AND_SEARCH.md](docs/05_ELASTICSEARCH_INDEXING_AND_SEARCH.md) — Mappings, search queries, and SQL fallback.
- [06_SLACK_OAUTH_AND_RATE_LIMIT_ALERTS.md](docs/06_SLACK_OAUTH_AND_RATE_LIMIT_ALERTS.md) — Slack Block Kit alerts and resiliency.
- [07_FRONTEND_DASHBOARD_AND_FIGMA_REPLICATION.md](docs/07_FRONTEND_DASHBOARD_AND_FIGMA_REPLICATION.md) — Figma design breakdown & component structure.
- [08_INTERVIEW_EXPLANATION_AND_QA_CHEAT_SHEET.md](docs/08_INTERVIEW_EXPLANATION_AND_QA_CHEAT_SHEET.md) — 2-minute pitch & answers to tough technical questions.

---

## 🚀 Quickstart Guide

### Prerequisites
- Node.js (v18+)
- Redis running on `localhost:6379` (or run `docker-compose up -d redis`)

### 1. Backend Setup
```bash
cd backend
npm install
npx prisma db push
npm run seed     # Populates sample data matching the Figma mockups
npm run dev      # Starts server on http://localhost:5000
```

### 2. Frontend Setup
In a new terminal:
```bash
cd frontend
npm install
npm run dev      # Starts React Vite app on http://localhost:3000
```

Visit `http://localhost:3000` in your browser!

### 3. Bull-Board Dashboard

Open `http://localhost:5000/admin/queues` to inspect live BullMQ queue metrics.
