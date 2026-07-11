# NoLoop — Delivery Roadmap & Agile Plan

**Goal: build the complete platform in 2 months (8 weeks).** Everything — separate sites per stakeholder, the WhatsApp patient bot, all four AI agents, RAG, fraud detection, UPI settlement, and a full test suite — ships inside this window.

This is an **intense but achievable** plan for a **team of 4 who are learning while building**. It stays doable three ways:

- **Each person owns a lane** so work runs in parallel.
- We **lean on managed services** (Supabase, Groq, WhatsApp Business API, hosted UPI) instead of building infrastructure.
- **Testing is built into every sprint**, not left until the end.

> **North Star (End of Week 8):**
>
> *Hospital submits → AI proofs & summarises → Rules & fraud check → Insurer approves → UPI settles → Patient receives WhatsApp updates*
>
> Everything is deployed and the complete test suite is green.

---

# 1. How We Work

## Cadence

| Ceremony | When | Duration | Purpose |
|----------|------|----------|---------|
| **Weekly Team Sync** | Every Friday | **15–20 min** | Progress updates, blockers, quick demo, plan next week |
| **Sprint** | Every 2 weeks | — | Four sprints across eight weeks |

---

## Weekly Team Sync (15–20 min)

Each member answers:

1. What did I complete this week?
2. What will I work on next week?
3. Any blockers or help needed?

If someone is blocked, pair immediately after the meeting instead of waiting another week.

Keep the meeting short and focused.

---

## Definition of Done

A story is considered **Done** only when it:

- Works end-to-end
- Includes appropriate **unit tests**
- Includes an **integration/E2E test** if multiple services are involved
- Passes CI
- Is reviewed by another teammate
- Is merged into `main`

No feature is "done" if testing is postponed.

---

## Team Ownership

Each person owns one primary lane.

| Owner | Responsibility |
|--------|----------------|
| **FE A** | Hospital Site + Admin Site |
| **FE B** | Insurer Site (+TPA) + Patient Dashboard |
| **Backend** | APIs, Database, Workflow, Rules Engine, Queue, UPI |
| **AI** | WhatsApp Bot, OCR, RAG, AI Agents, Fraud Detection |

Work happens **in parallel** every sprint.

If someone gets blocked, another teammate pairs with them until the blocker is removed.

---

## Learning While Building

Whenever using a new technology:

- Spend **a few hours** creating a small proof of concept.
- Learn the basics.
- Throw the prototype away.
- Build the real feature.

Prefer simple, documented solutions over clever ones.

---

# 2. Eight Week Roadmap

| Sprint | Weeks | Theme | Deliverables |
|---------|------|-------|--------------|
| **Sprint 1** | 1–2 | Foundation | Sites, database, rules engine, CI, WhatsApp setup |
| **Sprint 2** | 3–4 | Claim Flow | Submission workflow, OCR, first AI agent |
| **Sprint 3** | 5–6 | Intelligence | RAG, fraud detection, UPI settlement, patient dashboard |
| **Sprint 4** | 7–8 | Production Ready | Security, analytics, deployment, testing, launch |

---

# 3. Sprint 1 (Weeks 1–2)

## Foundation

| Story | Owner | Tests |
|-------|------|------|
| Split Hospital & Insurer into separate sites sharing one UI library | FE A/B | Smoke render tests |
| Login & JWT across all sites | FE | Authentication unit tests |
| Expand Prisma schema (Claim, Documents, Decisions, Fraud Flags, Events) | Backend | Migration verification |
| Seed demo hospitals, insurers, patients, policies | Backend | Seed validation |
| Rules Engine (coverage, exclusions, co-pay, limits) | Backend | Rule unit tests |
| WhatsApp sandbox message | AI | Sandbox send test |
| Setup Jest, Pytest & GitHub Actions | All | CI pipeline verification |

### Sprint 1 Goal

- Separate sites running
- Shared claim record working
- Rules engine operational
- CI pipeline green

---

# 4. Sprint 2 (Weeks 3–4)

## Claim Flow

| Story | Owner | Tests |
|-------|------|------|
| Hospital claim submission | FE A + Backend | Submission integration test |
| Workflow orchestrator (Submitted → Review → Approved/Rejected) | Backend | State transition tests |
| Query-Proofing Agent | AI | Agent evaluation tests |
| OCR document extraction | AI | Extraction tests |
| Insurer review queue & decisions | FE B + Backend | Decision persistence tests |
| Claim status tracking | FE A | UI tests |
| WhatsApp notifications | AI + Backend | Notification tests |

### Sprint 2 Goal

Complete claim journey:

Submit → AI validation → Review → Decision → Status updates → WhatsApp notification

---

# 5. Sprint 3 (Weeks 5–6)

## AI + Settlement

| Story | Owner | Tests |
|-------|------|------|
| Adjudication Assistant | AI | Summary tests |
| Confidence scoring | AI + Backend | Routing tests |
| Fraud Detection Agent | AI | Fraud evaluation |
| RAG over policy, billing & insurance knowledge | AI | Retrieval evaluation |
| Communication Agent (WhatsApp Q&A) | AI | Response evaluation |
| UPI settlement | Backend | Sandbox integration |
| Patient dashboard | FE B | UI tests |
| Async processing queue | Backend | Queue tests |

### Sprint 3 Goal

Every AI agent works.

RAG is grounded.

Payments settle.

Patient dashboard works.

**First complete End-to-End test passes.**

---

# 6. Sprint 4 (Weeks 7–8)

## Hardening & Launch

| Story | Owner | Tests |
|-------|------|------|
| TPA role | FE B + Backend | Access tests |
| Event Bus & Webhooks | Backend | Webhook integration |
| Redis cache | Backend | Cache tests |
| Analytics dashboards | FE + Backend | Metrics tests |
| White-label subdomains | Backend | Routing tests |
| Complete Test Pyramid | All | Full suite |
| AI evaluation | All | Accuracy tests |
| Performance testing | All | Load tests |
| Security review | All | Auth & tenancy verification |
| Docker & CI/CD | All | Deployment verification |
| Monitoring & Logging | All | Health checks |

### Sprint 4 Goal

Production-ready platform.

Everything deployed.

Everything tested.

Demo-ready.

---

# 7. Testing Strategy

| Level | Scope | Tools | Timeline |
|--------|------|------|----------|
| **Unit Tests** | Rules, Agents, Helpers | Jest, Pytest | Every sprint |
| **Integration Tests** | API + Database + AI | Jest, Pytest | Sprint 2 onwards |
| **E2E Tests** | Complete platform | Playwright | Sprint 3 onwards |
| **AI Evaluation** | Accuracy & RAG | Evaluation scripts | Sprint 3–4 |
| **Performance & Security** | Load, Auth, Multi-tenancy | k6, Locust, Manual | Sprint 4 |

### Testing Rules

- Every feature ships with tests.
- CI runs on every push.
- Failed tests block merges.
- Synthetic demo data keeps tests repeatable.
- The happy-path E2E test remains green from Sprint 3 onward.

---

# 8. Reality Check

This roadmap is ambitious but achievable if:

- Everyone owns their lane.
- Managed services are used instead of building infrastructure.
- The first version of every feature stays intentionally simple.
- Blockers are discussed during the **weekly sync** and resolved immediately afterward.

If time becomes tight, always protect the core demo flow:

**Hospital → AI → Insurer → UPI → WhatsApp**

Nice-to-have features such as advanced analytics, dedicated TPA site, and white-label support can be deferred.

---

# 9. Product Backlog

## Sites

- Hospital Site
- Insurer Site
- Admin Site
- Patient Dashboard
- WhatsApp Patient Bot
- White-label Subdomains

## Core Platform

- Shared Claim Record
- Rules Engine
- Workflow Orchestrator
- Queue
- Event Bus
- Webhooks
- Redis
- UPI Settlement
- Metrics Dashboard
- Admin CRUD

## AI

- Query-Proofing Agent
- Adjudication Assistant
- Fraud Detection Agent
- Communication Agent
- OCR
- RAG (3 Knowledge Bases)
- Confidence Scoring
- AI Evaluation Harness

## Testing & Operations

- Unit Tests
- Integration Tests
- End-to-End Tests
- AI Accuracy Testing
- Performance Testing
- Security Review
- Docker
- CI/CD
- Monitoring
- Demo Data

---

# 10. Legend

- **FE** – Frontend
- **BE** – Backend
- **AI** – AI & Integrations
- **All** – Entire Team

Owners are indicative and can be reassigned during the weekly team sync.
