# NoLoop

**One shared record. One fast decision. No loop.**

NoLoop is a multi-agent AI platform for health-insurance claim processing. It sits in
the middle of the ecosystem — **patient, hospital, TPA, and insurer** — on one shared
system, so a claim never has to bounce back and forth between three parties again.

Each stakeholder gets **their own website**, and the **patient is served through a
WhatsApp bot** (no app to install). A minimal patient web dashboard comes later.

---

## The problem — "The Loop"

When a patient is admitted and wants to use health insurance, the claim travels through
multiple hands:

```
Patient admitted → Hospital → TPA → Insurer → query back → Hospital → Insurer → query → …
```

This back-and-forth is **The Loop**. Three things make it painful:

1. **Documents are requested again and again.**
2. **No shared status** — three parties keep three separate files and three different
   pictures of the same claim.
3. **The patient pays cash to leave** — "cashless is pending, pay now, we'll refund
   later." *Later* becomes weeks, months, sometimes never.

Meanwhile the patient occupies a bed another emergency patient needs.

### The scale (India)

- **11–12.5%** of claims are denied, usually without a clear reason to the patient.
- **58%+** of claims are cashless (hospital paid directly) — the dominant, growing method.
- A clean cashless claim takes **2–3 hours**; a problematic one takes **days**.
- One human processor handles **50–80 claims/day**, by hand.
- **~15%** of claims involve fraud → **₹8,000–10,000 cr** in annual losses.
- Dec 2025: IRDAI penalised an insurer **₹1 cr** for processing delays and poor
  transparency. The regulator is watching.

---

## The solution

NoLoop is one shared platform — think a shared document every stakeholder can see —
that works in three steps:

| Step | What happens |
| ---- | ------------ |
| **1. Capture** | Hospitals submit structured claims. NoLoop catches missing info *before* submission, so claims aren't bounced back for errors. |
| **2. Decide**  | Policy rules (coverage, limits, exclusions) are auto-checked. High-confidence claims are decided instantly; the rest go to a human reviewer with an AI summary. |
| **3. Settle**  | The insurer pays the hospital directly (UPI). The patient pays only their co-pay and walks out. |

---

## Products — a separate site per stakeholder

Each stakeholder gets a focused product, deployed to its own subdomain. All sites share
one component library and talk to the same backend and shared claim record.

| Product | Who | What it does | Status |
| ------- | --- | ------------ | ------ |
| **Admin site** | Platform ops | Manage orgs & users, credentials, audit logs (god-mode) | Built (scaffold) |
| **Hospital site** | Billing desk | Submit claims/pre-auth, track status, manage beds/patients | Scaffold → MVP |
| **Insurer site** | Reviewing doctors + **TPA role** | Review queue, approve/override, TAT dashboards | Scaffold → MVP |
| **Patient — WhatsApp bot** | Policyholders | Real-time status updates + policy Q&A, no app to install | Planned |
| **Patient dashboard** | Policyholders | Minimal web view of the claim timeline | Later |
| **TPA site** | TPA teams | Dedicated TPA workspace *(only if the workflow needs it)* | Later / optional |
| **Landing site** | Public | Marketing + sign-up | Built |

> **Why not a separate TPA site from day one?** For a small team shipping fast, TPA is a
> *role inside the insurer site* to start with. It graduates to its own site only if the
> workflow genuinely demands separation.

### Three stakeholders, three benefits

- **Patient** — real-time **WhatsApp** updates at every step; no app to install; knows
  the exact co-pay before discharge.
- **Hospital** — one console for all insurers; file once, watch it clear; faster bed
  turnover.
- **Insurer / TPA** — clean structured claims instead of messy PDFs; AI-assisted review
  (target 3× reviewer throughput); every decision recorded and auditable.

---

## The four AI agents

NoLoop's intelligence is organised as four cooperating agents. **AI assists; a human
makes every final approval decision (human-in-the-loop).** *(All four are built within
the 8-week delivery plan — see the [roadmap](ROADMAP.md).)*

| Agent | Serves | Job |
| ----- | ------ | --- |
| **Query-Proofing Agent** | Hospital | Cross-checks documents against the policy *before* submission and flags mismatches instantly. |
| **Case Review (Adjudication) Agent** | Insurer | Reads medical files and summarises the case in plain English for the reviewing doctor. |
| **Fraud Detection Agent** | Insurer | Turns billing patterns and history into an explainable fraud-risk score. |
| **Communication Agent** | Patient | Powers the WhatsApp bot's updates and policy Q&A, citing exact policy clauses. |

---

## Architecture (overview)

```
   Clients:  Hospital site · Insurer site (+TPA) · Admin site · Patient WhatsApp bot
                                   │
                    ┌──────────────▼───────────────┐
                    │  Core Services (API Gateway)  │  auth/RBAC · Policy · Claim · Notification
                    └──────────────┬───────────────┘
                    ┌──────────────▼───────────────┐
                    │     Workflow Orchestrator     │  state machine over the claim lifecycle
                    └──────────────┬───────────────┘
                    ┌──────────────▼───────────────┐
                    │      Shared Claim Record      │  single source of truth
                    └──────────────┬───────────────┘
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
  AI & Rules Engine        Data Layer                  Integrations
  4 agents + RAG +         Postgres · Docs ·           UPI settlement ·
  Rules Engine             Redis · Audit · Analytics   WhatsApp Business API
```

Full layered design, claim state machine, and event flow: **[ARCHITECTURE.md](ARCHITECTURE.md)**.

---

## Tech stack

| Layer            | Technology                                                             |
| ---------------- | ---------------------------------------------------------------------- |
| Frontends        | Next.js / React, TypeScript, Tailwind CSS — one shared UI library, separate sites per stakeholder, white-labelled per client (e.g. `apollo.noloop.in`) |
| Backend API      | FastAPI (Python) — agent workflows; NestJS + Prisma service (current)  |
| AI layer         | Groq LLM via API · RAG over 3 knowledge bases (policy docs, clinical billing benchmarks, insurance rules) |
| Documents        | OCR + AI extraction (handles handwritten prescriptions)                |
| Database / cache | PostgreSQL (source of truth) · Redis (cache)                           |
| Messaging        | Event bus / message broker · Webhooks                                  |
| Patient comms    | WhatsApp Business API (bot)                                            |
| Settlement       | UPI payment rails                                                      |
| Infra            | Docker · CI/CD (GitHub Actions) · cloud deployment                     |

---

## Monorepo layout

**Today** the hospital, insurer, and patient views live as routes inside `noloop-app`;
the admin site is already separate. The roadmap splits them into their own deployable
sites (target layout shown with *→*).

```
noloop/
├── noloop-app/                 # Today: landing + hospital/insurer/patient routes + services
│   ├── app/                    #   → splits into noloop-hospital, noloop-insurer, patient dashboard
│   ├── components/ · lib/      #   → shared UI library used by all sites
│   ├── backend/                # NestJS + Prisma API (current primary)
│   │   ├── prisma/             #   schema.prisma + migrations
│   │   ├── scripts/            #   create-platform-admin, seed-demo, generate-synthetic-claims
│   │   └── src/                #   auth, admin, claims, beds, metrics, catalog, ai, health
│   ├── backend-py/             # FastAPI API (target backend for agent workflows)
│   └── ai/                     # FastAPI AI engine (pipeline: extract→coverage→validate→adjudicate→llm)
├── noloop-admin/               # Admin site (Next.js): orgs, users, logs
└── (planned) whatsapp-bot/     # Patient WhatsApp bot service
```

---

## Services & ports

| Service              | Path                     | Port   | Start command                    | Status  |
| -------------------- | ------------------------ | ------ | -------------------------------- | ------- |
| Landing / web        | `noloop-app`             | `3000` | `bun run dev`                    | Built   |
| Admin site           | `noloop-admin`           | `3001` | `bun run dev`                    | Built   |
| Hospital site        | `noloop-hospital` *(planned)* | `3002` | `bun run dev`               | Planned |
| Insurer site (+TPA)  | `noloop-insurer` *(planned)*  | `3003` | `bun run dev`               | Planned |
| WhatsApp bot         | `whatsapp-bot` *(planned)*    | `3005` | `bun run dev`               | Planned |
| Backend API          | `noloop-app/backend`     | `4000` | `bun run dev`                    | Built   |
| Backend (Py)         | `noloop-app/backend-py`  | `8001` | `uvicorn app.main:app --reload`  | Built   |
| AI engine            | `noloop-app/ai`          | `8000` | `./start.sh`                     | Scaffold|

---

## Getting started

### Prerequisites
Node.js 20+ and [Bun](https://bun.sh/) (or npm) · Python 3.11+ · a Supabase / PostgreSQL database.

### 1. Backend API + database
```bash
cd noloop-app/backend
bun install
cp .env.example .env            # set DATABASE_URL + JWT secret
bunx prisma migrate deploy      # apply migrations
bun run scripts/create-platform-admin.ts   # bootstrap the first admin
bun run scripts/seed-demo.ts               # seed demo orgs, policies, patients, beds, claims
bun run dev                     # → http://localhost:4000
```

### 2. AI engine
```bash
cd noloop-app/ai
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # set GROQ_API_KEY (optional; runs without it)
./start.sh                      # → http://localhost:8000
```

### 3. Frontends
```bash
cd noloop-app   && bun install && bun run dev     # → http://localhost:3000
cd noloop-admin && bun install && bun run dev     # → http://localhost:3001
```

> Quick reference: [`noloop-app/start.md`](noloop-app/start.md).

---

## Feature status

### Built
Landing page · sign-up & login · JWT auth · RBAC · multi-tenant architecture · custom
staff email generation · organization & employee management · admin dashboard, analytics
& audit trail · hospital & insurer portal scaffolds · backend API (NestJS + Prisma) +
FastAPI parity backend · Supabase integration · synthetic claim-data generator · AI
pipeline scaffold.

### Planned
Separate hospital / insurer sites · WhatsApp patient bot · workflow orchestrator (state
machine) · single shared claim record · rules engine · the 4 AI agents · OCR + RAG ·
confidence scoring · UPI settlement · patient dashboard · TPA site (optional) · Redis ·
event bus · webhooks · full test pyramid, security, CI/CD, monitoring, deployment.

See **[ROADMAP.md](ROADMAP.md)** for the full 8-week (2-month) delivery plan.

---

## Documentation

- **[ROADMAP.md](ROADMAP.md)** — the full 8-week (2-month) Agile plan for a 4-person
  learning team: every feature and its tests, sprint by sprint.
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — layered system design, state machine, event flow.

---

## How the vision maps to reality

This README describes the **full product vision**. The foundation — auth, multi-tenancy,
portals, and API/AI scaffolding — is built today; the separate sites, WhatsApp bot,
multi-agent intelligence, orchestration, settlement, and full test suite are delivered
across the **8-week roadmap**.

> **Human-in-the-loop:** NoLoop's AI never issues a final approval on its own — it reads,
> flags, summarises, and scores; a human reviewer signs off. Product claims such as
> "92% decided instantly" and "pre-auth in 8 minutes" are target outcomes, not measured
> production metrics.
