# NoLoop — System Architecture

This document describes NoLoop's target architecture end-to-end. Items marked
*(planned)* are on the [roadmap](ROADMAP.md); everything else describes the intended
design that the current codebase is growing toward.

> **Design principle:** one shared claim record, one orchestrated workflow, AI that
> *assists* and humans that *decide*. Every state change is auditable.

---

## 1. Layered overview

Each stakeholder has its **own website**; the patient is served by a **WhatsApp bot**.
All sites share one UI library and talk to the same backend + shared claim record.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ CLIENTS  (separate sites, one shared UI library)                              │
│  Hospital site   Insurer site (+TPA role)   Admin site   Patient WhatsApp bot │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                 │ HTTPS / JWT
┌───────────────────────────────▼───────────────────────────────────────────────┐
│ CORE SERVICES LAYER                                                           │
│  API Gateway  ·  RBAC / Auth  ·  Policy Service  ·  Claim Service  ·           │
│  Notification Service                                                          │
└───────────────────────────────┬───────────────────────────────────────────────┘
┌───────────────────────────────▼───────────────────────────────────────────────┐
│ WORKFLOW ORCHESTRATOR  (state machine — owns the claim lifecycle)             │
└───────────────────────────────┬───────────────────────────────────────────────┘
┌───────────────────────────────▼───────────────────────────────────────────────┐
│ SHARED CLAIM RECORD  (single source of truth — hospital = insurer = TPA view) │
└──────────────┬───────────────────────────────┬────────────────────────────────┘
               │                               │
┌──────────────▼────────────────┐   ┌──────────▼─────────────────────────────────┐
│ AI & RULES ENGINE             │   │ DATA LAYER                                 │
│  Query-Proofing Agent         │   │  PostgreSQL (SoT) · Document storage ·     │
│  Adjudication Assistant       │   │  Redis cache · Audit logs · Analytics DB   │
│  Fraud Detection Agent        │   └────────────────────────────────────────────┘
│  Communication Agent          │
│  Rules Engine  ·  RAG (3 KBs) │   ┌────────────────────────────────────────────┐
└───────────────────────────────┘   │ INTEGRATIONS                               │
                                     │  Hospital systems · Insurer systems ·      │
                                     │  UPI settlement · WhatsApp Business API    │
                                     └────────────────────────────────────────────┘
```

---

## 2. Clients — one site per stakeholder

Every stakeholder gets a focused, separately-deployed site (own subdomain), built from a
shared component library. The patient is reached through a WhatsApp bot, not a portal.

| Client | Users | Interface |
| ------ | ----- | --------- |
| **Hospital site** | Billing desk | Submit claims/pre-auth, track status, manage beds/admissions |
| **Insurer site** | Reviewing doctors, **TPA role** | Review queue, override, live TAT/metrics dashboards |
| **Admin site** | Platform ops | Org & user CRUD, credentials, audit logs (god-mode) |
| **Patient — WhatsApp bot** | Policyholders | Status updates + policy Q&A; no app install |
| **Patient dashboard** *(later)* | Policyholders | Minimal web view of the claim timeline |
| **TPA site** *(optional, later)* | TPA teams | Own workspace only if the workflow needs it |

Sites are **white-labelled** per client via subdomain *(planned)* — e.g.
`apollo.noloop.in` looks native to the hospital. TPA starts as a *role inside the insurer
site* and only becomes its own site if the workflow demands separation.

---

## 3. Core Services Layer

- **API Gateway** — single entry point for all requests: authentication, routing, rate
  limiting, logging. Services are never called directly from clients.
- **RBAC / Auth** — JWT-based; roles map to permissions (patient views only; hospital
  submits; insurer approves/rejects; admin manages users). Multi-tenant isolation.
- **Policy Service** — insurance rules: coverage, limits, exclusions, co-pay.
- **Claim Service** — claim CRUD and orchestration entry points.
- **Notification Service** — fans events out to WhatsApp, webhooks, and dashboards.

---

## 4. Workflow Orchestrator (state machine)

Every state change flows through the orchestrator rather than services calling each
other directly. This gives centralized control, easier monitoring, clean error
recovery, and scalability.

### Claim lifecycle

```
 SUBMITTED
    │  (Query-Proofing Agent validates documents)
    ▼
 VALIDATED ──► needs-info ──► RETURNED_TO_HOSPITAL ──┐
    │                                                │
    ▼                                                │ (resubmit)
 POLICY_CHECK  (Rules Engine: coverage/limits/excl.) │◄────────────┘
    │
    ▼
 FRAUD_CHECK  (Fraud Detection Agent → risk score)
    │
    ▼
 REVIEW  (Adjudication Assistant summary → human doctor decides)
    │
    ├──► APPROVED ──► SETTLEMENT ──► SETTLED
    └──► REJECTED  (reason recorded, patient notified)
```

### Why a state machine
A claim may move only through valid transitions. `SUBMITTED → REVIEW → APPROVED` is
valid; `SUBMITTED → SETTLED` is not. Invalid transitions are rejected, preventing
workflow corruption. Current enums (`ClaimStatus`, `Verdict`, `ClaimEventType`) in the
Prisma schema are the seed of this model.

---

## 5. Shared Claim Record (single source of truth)

Instead of a hospital record + insurer record + TPA record, there is **one** claim
record all parties read in real time. Benefits: no duplicate data entry, real-time
consistency, and simple auditing. Documents, status, decision, fraud score, and the
full event timeline all hang off this record.

---

## 6. AI & Rules Engine

### The four agents

1. **Query-Proofing Agent** *(hospital, pre-submission)* — validates documents against
   the policy; flags missing docs, wrong fields, policy mismatch before submission.
2. **Adjudication Assistant** *(insurer)* — extracts key medical details, matches policy
   rules, produces a plain-English case summary. **Human makes the final call.**
3. **Fraud Detection Agent** *(insurer)* — duplicate claims, abnormal billing, inflated
   costs, repeat indicators → an **explainable** fraud-risk score.
4. **Communication Agent** *(patient)* — drives WhatsApp updates and answers policy
   questions with citations to exact clauses.

### RAG (Retrieval-Augmented Generation)

```
User query / claim context
        │
        ▼
Retrieve from knowledge bases ──► (1) Policy documents
        │                         (2) Clinical billing benchmarks
        │                         (3) Insurance rules
        ▼
Provide retrieved context → LLM (Groq)
        │
        ▼
Grounded, citation-backed response
```

RAG keeps answers current and reduces hallucination — critical because policies change
often. The LLM is grounded in retrieved clauses rather than parametric memory alone.

### Rules Engine
Deterministic policy logic (coverage / limits / exclusions / co-pay). **AI + Rules**,
not AI alone: rules enforce policy; AI interprets unstructured documents.

### Today's pipeline
The current AI engine (`noloop-app/ai`) runs a typed pipeline
`extract → coverage → validate → adjudicate → LLM rationale`. This is the scaffold the
four agents and RAG grow out of.

---

## 7. Data Layer

| Store | Purpose |
| ----- | ------- |
| **PostgreSQL** | System of record — ACID, transactions, complex joins. Claims cannot tolerate inconsistency. |
| **Document storage** | Uploaded bills, reports, discharge summaries, prescriptions. |
| **Redis cache** *(planned)* | Sessions, policy lookups, hot claim status — cuts DB load and latency. |
| **Audit logs** | Immutable who/when/what for every change — regulatory requirement. |
| **Analytics DB** *(planned)* | Dashboards, TAT metrics, reporting. |

**Resilience:** replication, automated backups, and failover. Redis holds only
transient data; if the primary Postgres fails, traffic redirects to a replica while the
source of truth is restored.

---

## 8. Eventing & integrations

- **Event bus / message broker** *(planned)* — asynchronous, loosely-coupled fan-out.
  e.g. `Claim Approved → {Notification, Analytics, Settlement}` services react
  independently. Improves scalability and reliability.
- **Webhooks** *(planned)* — event callbacks into hospital/insurer systems
  (e.g. `Claim Approved → hospital HIS updated`).
- **UPI settlement** *(planned)* — insurer pays hospital directly on approval.
- **WhatsApp Business API** — patient notifications and Q&A.

---

## 9. Security & compliance

- HTTPS/TLS in transit; encryption at rest.
- JWT authentication + RBAC; multi-tenant isolation.
- Immutable audit trail on every action.
- Consent management; NHCX-aligned interoperability *(target)*.
- Human-in-the-loop on every final approval → accountability and regulatory alignment.

---

## 10. Deployment

Containerised with **Docker**, **CI/CD via GitHub Actions**, deployed to cloud infra
with monitoring and logging. No proprietary lock-in — components (LLM, cache, broker)
are swappable as the platform scales. See [ROADMAP.md](ROADMAP.md) Phase 6.
