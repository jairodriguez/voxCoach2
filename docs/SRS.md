# Software Requirements Specification – **VoxCoach Platform**

### Prepared for Firebase Studio implementation

*Last updated: 27 Apr 2025*

---

## 1  Introduction

### 1.1 Purpose

Provide a complete, implementation‑ready specification so a Firebase Studio team can build the **VoxCoach** multi‑tenant, voice‑first coaching SaaS (front‑end + back‑end + infra) with confidence and traceability.

### 1.2 Scope

Covers MVP through Public Launch (Phases 0‑2) including: authentication, multitenancy, Supabase knowledge‑base storage & vector search, conversational coaching flows, adaptive upsells, certification system, Stripe Connect split payments, dual storage targets (Supabase Storage and Google Drive), dashboards, and admin tooling.

### 1.3 Definitions / Glossary

- **Tenant** – A business/customer running its own branded coaches.
- **Learner** – End user receiving coaching.
- **Phase** – Discrete coaching stage (Foundation, Engagement, etc.).
- **Agent** – OpenAI real‑time voice bot (Coach, Sales, Teacher).
- **RAG** – Retrieval‑Augmented Generation: embedding tenant docs for context.
- **Certificate ID** – Verifiable ID in format `TENANT‑PHASE‑YYMM‑#####`.

### 1.4 References

| Ref ID   | Resource                                                       |
| -------- | -------------------------------------------------------------- |
| **R‑01** | VoxCoach PRD v0.9 (textdoc `680e779cff88819181ba740941ceac90`) |
| **R‑02** | **Supabase Docs – Platform & JS Client** (supabase.com/docs)   |
| **R‑03** | **Supabase Vector Extension** (beta) – pgvector & search RPC   |
| **R‑04** | **Firebase Studio** – AI‑assisted IDE for Firebase projects    |
| **R‑05** | OpenAI Realtime Agents GitHub repo (fork)                      |
| **R‑06** | Stripe Connect & Checkout API docs                             |
| **R‑07** | Google Drive REST v3 API docs                                  |

### 1.5 Overview

Sections 2–14 specify functional & non‑functional requirements, data models, APIs, UI guidelines, and DevOps workflows.

---

\## 2  Overall Description

### 2.1 Product Perspective

Full‑stack application built on **Next.js 14** using Vercel’s official **SaaS Starter Kit** (App Router, TypeScript, Tailwind, Clerk-ready). Deployed on the Vercel Edge Network. Server Actions and Edge Functions handle real‑time websockets to OpenAI’s end‑point and call Supabase RPC for vector search.

- **Tenant Routing :** each tenant is served from an automatic wildcard sub‑domain pattern `https://{tenant}.voxcoach.app`, configured via Vercel wildcard domains and Next.js middleware to inject `tenant_id` at runtime.

Embeddings, user data, and RLS remain in Supabase Postgres with pgvector.

### 2.2 Product Functions Product Functions

1. Tenant onboarding + Stripe Connect.
2. Knowledge‑base upload → RAG embedding in Supabase.
3. Voice coaching sessions with agent hand‑offs.
4. Personalized doc generation & dual‑storage.
5. Quiz engine + certificate PDF issue.
6. Adaptive cross‑sell & checkout.
7. Gamified learner dashboard.
8. Admin analytics & overage billing.

### 2.3 User Classes & Characteristics

| Class              | Expertise                      | Needs                                       |
| ------------------ | ------------------------------ | ------------------------------------------- |
| **Tenant Admin**   | Non‑technical coach/consultant | Upload docs, set brand, pricing, phases.    |
| **Learner**        | End user                       | Seamless voice session, docs, certificates. |
| **Platform Admin** | Jairo / ops                    | Manage tenants, pricing models, usage caps. |

### 2.4 Operating Environment

- Supported browsers: latest Chrome, Edge, Safari, Firefox.
- Audio: WebRTC microphone; fallback to text chat.
- Deployment: Vercel Serverless Functions (Node 20 runtime).

### 2.5 Constraints

- PCI compliance handled via Stripe Checkout only (no raw card capture).
- Google Drive API quota 10 k requests/day → batching uploads.

### 2.6 Assumptions & Dependencies

- OpenAI realtime voice endpoints remain stable.
- Supabase Starter tier adequate for ≤ 1 M vectors initially.

---

\## 3  System Features & Functional Requirements

> *Each feature tagged **``** for traceability.*

### 3.1 Multitenancy (`REQ‑F1`)

- **F1‑1** Create Tenant row in Supabase (`tenants` table) on signup; roles: `OWNER`, `ADMIN`, `ANALYST`.
- **F1‑2** Supabase RLS policies enforce `tenant_id` segregation.
- **F1‑3** Automatic wildcard sub‑domain per tenant (`{tenant}.voxcoach.app`) via Vercel DNS; optional custom domain mapping.

### 3.2 Authentication (`REQ‑F2`)

- **F2‑1** Support Google OAuth & magic‑link email via Supabase Auth.
- **F2‑2** Next.js (App Router) front‑end retrieves Supabase JWT (including `tenant_id`, `role` claims) via the Supabase JS client.
- **F2‑3** Refresh claims on role change.

### 3.3 Knowledge‑Base Upload & RAG (`REQ‑F3`)

- **F3‑1** Drag‑and‑drop uploader → Supabase Storage `tenants/{tenantId}/kb/`.
- **F3‑2** Storage webhook extracts text, chunks, calls OpenAI embeddings, stores vectors in `kb_vectors` table.
- **F3‑3** Delete trigger removes vectors.

### 3.4 Voice Coaching Session (`REQ‑F4`)

- **F4‑1** Front‑end opens WebSocket `/realtime` channel.
- **F4‑2** Back‑end routes messages to active agent persona.
- **F4‑3** Agent can emit `handoff` event.
- **F4‑4** Session transcript stored in `sessions` table.

### 3.5 Document Generation (`REQ‑F5`)

- **F5‑1** Agent triggers `generateDoc` function with template + vars.
- **F5‑2** Node PDF generator outputs PDF to Supabase Storage `users/{userId}/docs/`.
- **F5‑3** If learner granted Drive scope, additional upload → `VoxCoach Docs` folder.

### 3.6 Quiz & Certificate (`REQ‑F6`)

- **F6‑1** Teacher agent fetches quiz JSON or GPT‑generated set.
- **F6‑2** Score ≥ threshold → `issueCert` function.
- **F6‑3** Certificate PDF created from tenant template, ID embedded, stored + emailed.

### 3.7 Adaptive Cross‑Sell (`REQ‑F7`)

- **F7‑1** Analyze `sessions.meta` tags.
- **F7‑2** Sales agent offers discount.
- **F7‑3** `createCheckoutSession` CF creates Stripe Checkout; returns `redirect_url`.

### 3.8 Gamification (`REQ‑F8`)

- **F8‑1** XP rules configurable per tenant.
- **F8‑2** `xp_log` table records events.
- **F8‑3** Dashboard shows progress bar & badge.

### 3.9 Analytics (`REQ‑F9`)

- **F9‑1** Daily scheduled CF aggregates token, storage, revenue into `tenant_stats`.
- **F9‑2** Optional BigQuery export.

---

\## 4  External Interface Requirements

### 4.1 User Interface Overview

| Route               | Components                                          | Notes             |
| ------------------- | --------------------------------------------------- | ----------------- |
| `/login`            | AuthCard (Supabase UI)                              |                   |
| `/dashboard`        | ProgressRing, CertList, UpsellBanner                | Learner view      |
| `/session/:phaseId` | ConversationPane, VoiceVisualizer, ResourceSidebar  |                   |
| `/admin`            | PhaseTable, ResourceUploader, BrandForm, PricingTab | Tenant admin      |
| `/verify/:certId`   | CertLookupForm, CertCard                            | Public validation |
| `/super`            | TenantGrid, UsageCharts                             | Platform admin    |

### 4.2 UI Library & Design System

| Element   | Choice                                                |
| --------- | ----------------------------------------------------- |
| Framework | **Next.js 14 (Vercel SaaS Starter Kit)** + TypeScript |
| Styling   | Tailwind + ShadCN UI                                  |
| Font      | Inter, Roboto Mono                                    |
| Palette   | `#1F6FEB`, `#0A0F1A`, `#FF9F1C`, `#F3F5FA`, `#2E3545` |
| Radius    | `rounded‑2xl`                                         |
| Shadow    | `shadow‑lg`                                           |
| Motion    | Framer‑motion                                         |

### 4.3 Software Interfaces

- **Stripe** – Checkout + webhooks.
- **OpenAI Realtime** – Audio chat.
- **Supabase JS** – Auth, Postgres CRUD, Storage, RPC (vector search).
- **Google Drive REST** – File uploads.

### 4.4 Comms Interfaces

- HTTPS (REST) for CF; WebSocket for realtime agent stream.

---

\## 5  Architecture & Design *(High‑level diagram unchanged from PRD; now notes Supabase tables instead of Firestore collections.)*

### 5.1 Vercel Functions (Node 20)

| Function                | Trigger                  | Purpose |
| ----------------------- | ------------------------ | ------- |
| `createCheckoutSession` | callable HTTPS           |         |
| `stripeWebhook`         | HTTPS endpoint           |         |
| `onKbUpload`            | Supabase Storage webhook |         |
| `issueCert`             | callable                 |         |
| `dailyStats`            | scheduled                |         |

### 5.2 Supabase Schema (core)

```
tenants, users, phases, quizzes, sessions, docs, certificates,
kb_vectors, xp_log, tenant_stats, payments
```

Row‑level security ensures `tenant_id = auth.jwt().tenant_id`.

### 5.3 Storage Buckets

```
/tenants/{tenantId}/kb/**
/users/{userId}/docs/**
```

---

\## 6  Non‑Functional Requirements

| Category    | Requirement                                                      |
| ----------- | ---------------------------------------------------------------- |
| Performance | <200 ms p95 latency per agent response (ex‑OpenAI stream).       |
| Scalability | 10 k concurrent sessions (Supabase + Cloud Functions autoscale). |
| Reliability | 99.5 % uptime SLA.                                               |
| Security    | Supabase RLS + JWT, Stripe PCI scope via Checkout.               |
| Privacy     | PII minimal.                                                     |

---

\## 7  Dev & Deployment Workflow

1. Mono‑repo scaffolded from **Vercel SaaS Starter Kit** (Turborepo + Next.js).
2. GitHub Actions → Vercel preview deployments.
3. Supabase database migrations via `supabase migrate`.
4. Prod deploy on `main`.

---

\## 8  Open Issues & Future Work

- Firebase Vector extension once GA.
- Multi‑language voices.
- PWA offline viewer.

---

> **End of SRS v1.1** – Supabase references added; all prior design decisions preserved.

