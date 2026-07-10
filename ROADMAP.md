# NoLoop — Delivery Roadmap & Agile Plan

**Goal: build the complete platform in 2 months (8 weeks).** Everything — separate sites
per stakeholder, the WhatsApp patient bot, all four AI agents, RAG, fraud detection, UPI
settlement, and a full test suite — ships inside this window.

This is an **intense but achievable** plan for a **team of 4 who are learning while
building**. It stays doable three ways: **each person owns a lane** so work runs in
parallel, we **lean on managed services** (Supabase, Groq, WhatsApp Business API,
hosted UPI) instead of building infra, and **testing is woven into every sprint** — not
saved for the end.

> **North star (end of Week 8):** a claim runs the full journey live —
> *Hospital submits → AI proofs & summarises → rules + fraud check → insurer approves →
> UPI settles → patient tracked on WhatsApp* — with tests green and the app deployed.

---

## 1. How we work

### Cadence

| Ceremony            | When                | Duration  | Purpose                              |
| ------------------- | ------------------- | --------- | ------------------------------------ |
| **Daily standup**   | Every working day   | 10–15 min | Yesterday / today / blockers         |
| **Weekly catch-up** | Every Friday        | 45–60 min | Demo, retro, plan the coming week    |
| **Sprint = 2 weeks**| —                   | —         | 4 sprints across the 8 weeks         |

### Standup: 3 questions
1. What did I finish?  2. What am I doing today?  3. What's blocking me?
Blocked > 2 hours → pair up immediately. With this timeline, nobody stays stuck.

### Weekly catch-up
Demo what works → quick retro (well / hard / one change) → plan next week.

### Definition of Done — **tests are part of Done**
A story is done when it: works when clicked through · **has unit tests for its logic** ·
**has an integration/E2E test if it crosses services** · is merged to main · passes CI ·
was reviewed by a teammate. No "we'll test it later."

### Team of 4 — one lane each (pair across lanes when blocked)
- **FE A** — Hospital site + Admin site
- **FE B** — Insurer site (+ TPA role) + Patient dashboard
- **BE** — API, shared claim record, orchestrator, rules engine, UPI, queue
- **AI** — WhatsApp bot, the 4 AI agents, OCR, RAG, fraud

Lanes run in parallel every sprint. Swap a task occasionally so everyone learns the
whole system — but on this timeline, depth-in-your-lane comes first.

### Learning without losing time
Do a **short spike first** (a few hours, not days) when a tool is new — a quick tutorial
or throwaway proof of concept — then build for real. Prefer the boring, well-documented
option over the clever one.

---

## 2. The 8-week map

| Sprint | Weeks | Theme | Ships |
| ------ | ----- | ----- | ----- |
| **1** | 1–2 | Sites, data model, rules, CI | Separate sites live, shared claim record, rules engine, WhatsApp "hello", test+CI setup |
| **2** | 3–4 | Claim flow + first agents | Submit → orchestrated review → approve/reject, Query-Proofing Agent, OCR, status tracking, WhatsApp notifications |
| **3** | 5–6 | Full intelligence + settlement | Adjudication + Fraud agents, RAG, confidence scoring, UPI settlement, patient dashboard, two-way WhatsApp |
| **4** | 7–8 | Complete, harden, launch | TPA, event bus, Redis, analytics, white-label, full test pyramid, security, Docker, CI/CD, deploy |

Phase 0 (already done): auth, RBAC, multi-tenant, portal scaffolds, backends, AI scaffold.

---

## 3. Sprint 1 (Weeks 1–2) — Sites, data model, rules, CI

| Story | Owner | Tests |
| ----- | ----- | ----- |
| Split hospital & insurer into their own sites sharing one UI library (admin already separate) | FE A/B | Smoke render tests |
| Login/JWT working across all sites | FE | Auth unit tests |
| Expand Prisma schema: claim, documents, decision, fraud flag, events (shared claim record) | BE | Schema/migration check |
| Migrate + seed demo orgs, policies, patients, beds, historical claims | BE | Seed sanity test |
| **Rules engine** (coverage / limits / exclusions / co-pay) | BE | Unit tests per rule |
| WhatsApp bot: send one test message (sandbox) | AI | Sandbox send test |
| **Set up testing + CI**: Jest (FE/Node), pytest (Python), GitHub Actions runs tests on push | All | The pipeline itself |

**End of Sprint 1:** all sites run, a claim record exists, rules evaluate, CI is green.

---

## 4. Sprint 2 (Weeks 3–4) — Claim flow + first agents

| Story | Owner | Tests |
| ----- | ----- | ----- |
| Hospital site: claim submission form → saves to shared record | FE A / BE | Integration: submit persists |
| **Workflow orchestrator** (state machine): Submitted → Review → Approved/Rejected | BE | State-transition unit tests (reject illegal moves) |
| **Query-Proofing Agent** (LLM checks docs vs policy before submit) | AI | Agent unit tests + fixtures |
| OCR + structured extraction from uploaded documents | AI | Extraction accuracy test |
| Insurer site: review queue + approve / reject / **override** with reason | FE B / BE | Integration: decision persists |
| Hospital site: track claim status | FE A | UI + read test |
| WhatsApp notifications fire on status change | AI / BE | Notification-hook test |

**End of Sprint 2:** a claim goes submit → AI-proofed → insurer decides → status tracked
→ patient notified. Integration test covers the happy path.

---

## 5. Sprint 3 (Weeks 5–6) — Full intelligence + settlement

| Story | Owner | Tests |
| ----- | ----- | ----- |
| **Adjudication Assistant**: plain-English case summary for insurer | AI | Summary unit tests |
| **Confidence scoring** → auto-decide high confidence, route rest to human | AI/BE | Threshold/routing tests |
| **Fraud Detection Agent**: explainable risk score | AI | Fraud-rule tests + labelled set |
| **RAG** over 3 knowledge bases (policy docs, billing benchmarks, insurance rules) | AI | RAG eval harness (accuracy vs answers) |
| **Communication Agent**: two-way WhatsApp Q&A ("Is my implant covered?") with clause citations | AI | Q&A eval tests |
| **UPI settlement** (insurer → hospital on approval) | BE | Settlement integration test (sandbox) |
| Patient **dashboard** (minimal claim timeline) | FE B | UI + read test |
| Queue for async AI processing | BE | Queue job test |

**End of Sprint 3:** every AI agent works, RAG grounds answers, approvals settle over
UPI, patient sees a timeline. **First full end-to-end (E2E) test passes.**

---

## 6. Sprint 4 (Weeks 7–8) — Complete, harden, launch

| Story | Owner | Tests |
| ----- | ----- | ----- |
| TPA role in insurer site (dedicated site only if time allows) | FE B / BE | Role-access tests |
| Event bus + webhooks into hospital/insurer systems | BE | Event fan-out + webhook tests |
| Redis cache (sessions, policy lookups, status) | BE | Cache hit/miss test |
| Analytics / TAT & metrics dashboards | FE/BE | Metrics endpoint tests |
| White-label subdomains (`apollo.noloop.in`) | BE | Routing test |
| **Full test pyramid**: unit + integration + **E2E** (submit → decide → settle → notify) | All | The suite itself |
| **Accuracy** eval on AI · **performance** (load) · **security** review (authz, tenancy, secrets, consent) | All | Eval + load + security checks |
| Dockerize all services + CI/CD deploy on merge | All | CI build + deploy test |
| Monitoring & logging + final demo walkthrough | All | Health checks |

**✅ End of Sprint 4 (Week 8):** the complete platform is built, tested green across the
pyramid, deployed, and demo-ready.

---

## 7. Testing strategy (woven through, not bolted on)

| Level | What | Tools | When |
| ----- | ---- | ----- | ---- |
| **Unit** | Rules, agents, state transitions, helpers | Jest (FE/Node), pytest (Python) | Every story, every sprint |
| **Integration** | API ↔ DB ↔ AI engine, submit→decide→settle | Jest/pytest + test DB | Sprints 2–4 |
| **E2E** | Full user journey across sites + bot | Playwright (web), scripted bot test | Sprints 3–4 |
| **AI eval** | Agent accuracy vs a labelled synthetic set | `ai/scripts/eval.py` + RAG eval | Sprints 3–4 |
| **Perf / security** | Load on endpoints/queue; authz, tenancy, secrets, consent | k6/locust + manual review | Sprint 4 |

**Rules:** tests ship in the same PR as the feature · CI (GitHub Actions) runs the suite
on every push and blocks merges on red · seed/synthetic data drives repeatable tests ·
the E2E happy path must stay green from Sprint 3 onward.

---

## 8. Reality check

This delivers the full vision in 8 weeks — it is ambitious. It works **only** if:
- Each person stays in their lane so four features progress at once.
- We use managed services (Supabase, Groq, WhatsApp API, hosted UPI) — don't build infra.
- Scope stays lean: the simplest version of each feature first, polish later.
- Blockers are surfaced in the daily standup and cleared by pairing, same day.

If a sprint slips, **protect the E2E demo flow first** (submit → decide → settle →
notify) and defer the nice-to-haves (white-label, dedicated TPA site, advanced
analytics) — they're marked as the flexible items above.

---

## 9. Backlog (single source of truth)

**Sites & clients** — hospital site · insurer site (+TPA role) · admin site · patient
WhatsApp bot · patient dashboard · white-label subdomains.

**Core & orchestration** — shared claim record · rules engine · workflow orchestrator
(state machine) · queue · event bus · webhooks · Redis · UPI settlement · role-scoped
metrics · admin CRUD.

**AI** — Query-Proofing Agent · Adjudication Assistant · Fraud Detection Agent ·
Communication Agent · OCR · RAG (3 KBs) · confidence scoring · eval harness.

**Testing & ops** — unit · integration · E2E · AI accuracy · performance · security ·
Docker · CI/CD · monitoring · demo data.

---

## 10. Legend

**FE** Frontend · **BE** Backend · **AI** AI / integrations · **All** whole team.
Owners are indicative — assign real names at each Friday catch-up.
