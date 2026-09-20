# TJSA — Golang End-to-End Backend Implementation Plan

**Companion to:** `Autonomous-Banking-AI-Agent-BRD-PRD-HLD-LLD-Blueprint.md` v2.3 (the Blueprint). Design authority stays with the Blueprint. This document is the **backend engineering execution plan**: what to code in Go, in what order, which deployable owns it, and which business / functional / non-functional requirement it satisfies.

**Estate (Volume X, binding):** existing RBI-compliant mobile + IB apps; GCP multi-cluster GKE + ASM/Istio; edge GLB → FW → WAF → ILB → NGINX → ALB → Istio; Cloud Logging + Pub/Sub → BigQuery; 15+ crore envelopes in ~12 daytime hours; **12,000 events/s peak**.

**Plan version:** 2.2 — CQRS Query/Command planes + RAG as LLM knowledge base (aligned with Blueprint v2.4); backend-only start; full Volume III register mapped.

---

## 0. Does this plan contain all BR / FR / NFR?

**v1.0 did not.** It named services and phases but omitted the requirements register. **v2.0 does**, for the **backend**. Coverage rule:

| Source | Count | How this plan treats them |
|---|---|---|
| BR-001…014 | 14 | Every business outcome is a phase capability with an exit gate |
| PR-001…018 | 18 | Backend API + conversation contract; RN/React *screens* are channel work (thin) — this plan owns the APIs those screens call |
| FR-001…024 | 24 | Implemented as tools, state machines, or gateway rules |
| NFR-001…017 | 17 | SLO, security, DR, degradation — coded and load-tested |
| AI-001…015 | 15 | Model gateway, planner, RAG, evaluation gates |
| AG-001…015 | 15 | Orchestrator + tool-gateway + action-workflow |
| DATA-001…020 | 20 | PG/Redis/BQ/Neo4j schemas and lifecycle jobs |
| API-001…012 | 12 | Catalogue, taxonomy CI, tool contracts |
| UI-001…014 | 14 | UI registry + Task Guidance Engine (backend); extractors are CI |
| OBS-001…012 | 12 | Correlation, evidence APIs, dashboards |
| SEC-001…015 | 15 | Gateway, IAM, PII, kill switch, red team |
| GOV-001…012 | 12 | Registries, maker-checker, release gates |
| QA-001…013 | 13 | Test suites that block CI / release |
| OPS-001…015 | 15 | Runbooks, consoles, cost, learning loop |
| CP-001…011 | 11 | Tokenisation, residency, no-training, DLP |

**227 IDs.** The master map is §11. Channel-only work (RN/React chat sheet, CTA, deep-link handler) is called out as **channel**, not omitted. Everything else is backend Go (or a CronJob / Node extractor that feeds the backend).

**Also covered from the Blueprint (not extra IDs — they collapse onto the register):**

| Blueprint item | Maps to plan |
|---|---|
| BRD goals **G1–G10** (I.4) | G1→FR-006/AI-010; G2→BR-012; G3→BR-004; G4→BR-006/AG-011; G5→NFR-007/008; G6→FR-008/017; G7→BR-013/OPS-014; G8→BR-008; G9→FR-014/OPS-015; G10→PR-015/FR-021 |
| S1 **FR-1…FR-12** (III.0 crosswalk) | FR-1→FR-003; FR-2→FR-004+005; FR-3→FR-008+AI-002; FR-4→FR-011; FR-5→FR-012; FR-6→FR-013; FR-7→FR-014; FR-8→PR-001/002+UI-001/002; FR-9→BR-012; FR-10→FR-015; FR-11→AG-011; FR-12→AG-009+DATA-006 |
| II.9 NFR table | Same NFR-001…017 |
| Volume X estate (12k/s, Pub/Sub, BQ, edge) | Phases 0–2 evidence/ingest design |

**What this file is not:** a second copy of the Blueprint. Full requirement *text*, acceptance criteria, field schemas, and prompts stay in the Blueprint. This file is the ordered **backend build** that implements them.

---

## 1. Complete logical-service inventory

Everything is Go except one Node build tool. One monorepo (`tjsa/`), hexagonal layout per Blueprint V.2 (`cmd/<service>` + shared `internal/`). Logical services below; **deployables** are in §1a.

| # | Service | Binary | Purpose (Blueprint) | Build phase | Key dependencies |
|---|---|---|---|---|---|
| 1 | `agent-gateway` | `cmd/agent-gateway` | Conversation API, OIDC session, SSE, rate limits (V.1) | 1 | Existing IDP, Redis, orchestrator |
| 2 | `agent-orchestrator` | `cmd/agent-orchestrator` | State machine, planner, tool sequencing, response composer (§25, V.10) | 1 | tool-gateway, model-gateway, policy, Redis, PG |
| 3 | `tool-gateway` | `cmd/tool-gateway` | Tool registry, JSON Schema, authz, kill switch, audit (V.8) | 0 | policy-service, domain services |
| 4 | `model-gateway` | `cmd/model-gateway` | LLM adapters, PII guard, budgets, structured output (V.13) | 0 → 1 | prompt-registry, KMS, LLM |
| 5 | `prompt-registry` | `cmd/prompt-registry` | Versioned prompts, locale bundles (V.14) | 0 | PG |
| 6 | `policy-service` | `cmd/policy-service` | Embedded OPA, `Check` API (§30, V.9) | 0 | PG |
| 7 | `transaction-intelligence` | `cmd/transaction-intelligence` | `resolveJourney`, `getJourneyTrace`, `getFinancialState`, RCA, state engine (§20, §24) | 0 → 1 | evidence-service, PG, core status APIs |
| 8 | `evidence-service` | `cmd/evidence-service` | Curated evidence over BQ + Logging + Redis (X.5) | 0 | BQ, Cloud Logging, Redis, PG |
| 9 | `taxonomy-service` | `cmd/taxonomy-service` | Rules lookup, templates, gap logging (§23) | 0 | PG, Redis |
| 10 | `knowledge-service` | `cmd/knowledge-service` | **RAG knowledge base for the LLM** — retrieve/filter/pack/cite (Blueprint §16.3); never financial truth | 0 (schema/ingest) → 1 (live) | PG + pgvector, Neo4j |
| 11 | `ui-guidance-service` | `cmd/ui-guidance-service` | Deep links, Task Guidance Engine, prerequisites, rail advisor (§19) | 1 | PG, Redis, Pub/Sub, tool-gateway |
| 12 | `case-service` | `cmd/case-service` | Idempotent tickets, adapters (§28) | 1 | PG, ticketing |
| 13 | `action-workflow` | `cmd/action-workflow` + `cmd/action-worker` | Action Registry, Temporal, idempotency ledger (§27, V.12) | 2 | Temporal, PG, step-up, bank APIs |
| 14 | `journey-intelligence` | `cmd/journey-intelligence` | Journey model, graph queries (§18) | 2 | Neo4j, PG |
| 15 | `incident-service` | `cmd/incident-service` | Incident registry, impact correlation (§39) | 2 | PG, IMS |
| 16 | `audit-service` | `cmd/audit-service` | Hash-chained append-only audit (V.7) | 0 | PG (WORM), GCS |
| 17 | `admin-api` | `cmd/admin-api` | Config admin, kill switches, consoles (§53–54) | 0 → 4 | PG, audit |
| 18 | `kg-ingest` | `cmd/kg-ingest` | OpenAPI/Java → registries/graph (§37.4) | 0 batch | GCS, PG, Neo4j |
| 19 | `code-intelligence-indexer` | `cmd/code-intelligence-indexer` | AST + OpenAPI + UI nav → index (§38) | 0 batch | Repos read-only |
| 20 | `evidence-ingest` | `cmd/evidence-ingest` | Filtered Pub/Sub → T-2 hot store (X.5.1) | 2 | Pub/Sub, PG |
| 21 | `ui-knowledge-publisher` | Node CLI | RN/React routes → UI registry (§19) | 0 | App repos, PG |

**Shared libraries (Phase 0, before any service):**

| Package | Contents | Requirements |
|---|---|---|
| `internal/platform/otel` | `x-journey-id` / `traceparent` extract → log → propagate | OBS-001, OBS-011 |
| `internal/platform/logging` | `slog` JSON schema, PII-safe fields | OBS-011, OBS-005, DATA-010 |
| `internal/platform/pubsub` | Pub/Sub wrappers, ordering keys, idempotent upsert by `event_id` | OBS-004, DATA-004 |
| `internal/platform/pg` | pgx, sqlc, golang-migrate, partition-drop | DATA-009 |
| `internal/platform/httpx` / `rpc` | chi + Connect-Go, health, auth middleware | SEC-001, SEC-011 |
| `internal/platform/tokenizer` | HMAC-SHA256 searchable tokens, AES-256-GCM via Cloud KMS | NFR-006, DATA-010, CP-002 |
| `internal/domain` | Journey, TransactionEvent, EvidenceObject, TaxonomyEntry, Action, Investigation | DATA-001, DATA-002, DATA-006, DATA-012 |
| `internal/evidence` | Parameterised BQ templates + `maximum_bytes_billed`, Logging reader, allow-list | OBS-006, FR-006, X.5 C5 |

Standard libraries (Blueprint V.2): `chi`, `connectrpc.com/connect`, `pgx/v5` + `sqlc`, `go-redis/v9`, `cloud.google.com/go/{pubsub,bigquery,logging,kms}`, `neo4j-go-driver/v5`, `go.temporal.io/sdk`, `opa/rego`, `jsonschema/v6`, `anthropic-sdk-go` (Vertex/Bedrock adapters), `testify`, `testcontainers-go`.

## 1a. Consolidated deployment topology (recommended)

21 logical services → **7 deployables + CronJobs**. Hexagonal monorepo: merging/splitting later is a new `main.go`, not a rewrite.

**Never club (security boundaries):**

| # | Rule | Why |
|---|---|---|
| B1 | `model-gateway` always alone | Only process with LLM egress and **zero** data-store IAM (X.3) — NFR-006, SEC-012, CP-002 |
| B2 | `tool-gateway` never shares a process with `agent-orchestrator` | Enforcement choke point — AG-002, AI-011, SEC-002 |
| B3 | Audit **writer** isolated | Hash-chained WORM must not share failure domain — AG-009, DATA-015, NFR-008 |

| Deployable | Clubs | Phase |
|---|---|---|
| `agent-api` | agent-gateway + agent-orchestrator | 1 |
| `tool-plane` | tool-gateway + OPA embedded + prompt-registry | 0 |
| `model-gateway` | alone (B1) | 0 |
| `intelligence` | transaction-intelligence + journey-intelligence + evidence-service + taxonomy-service. `evidence-ingest` = same image, separate Deployment | 0 (ingest: 2) |
| `guidance` | knowledge-service (**RAG KB for LLM**) + ui-guidance-service | RAG + deep links + task procedures; Query plane only | 0–1 |
| `actions` | action-workflow + worker + case-service + incident-service | 2 |
| `governance` | admin-api + audit **read/export**; audit **writer** separate (B3) | 0 |
| CronJobs | kg-ingest, code-intelligence-indexer, learning-pipeline, ui-knowledge-publisher | 0 / 5 |

Phase 0–1 = 6 deployables; Phase 2 adds `actions` + `evidence-ingest`.

---

## 2. Phase plan at a glance

| Phase | Weeks | Autonomy | Backend deliverables | Gate owner |
|---|---|---|---|---|
| 0 Foundation | 8–12 | — | Platform skeleton, **CQRS `/q`+`/c` stubs**, correlation, taxonomy UPI+IMPS, evidence lazy path, **RAG schema+ingest**, audit, policy, tool-gateway | CTO / EA |
| 1 Explain-only MVP | 12–16 | L0 | **Query plane** EXPLAIN+GUIDE+INFORM + **RAG live**; **Command** tickets/escalate; payments rails | Digital Banking + Risk |
| 2 Safe actions | 12–20 | L1 | **Command plane** Action Registry + Temporal, T-2 ingest, incidents, cards/beneficiaries/mandates | Risk + Security |
| 3 Domain expansion | 12–14 | L1 | Deposits, loans, KYC, disputes — **content + tools, no new services** | Product + Compliance |
| 4 Contact-centre copilot | 8–10 | L1 | Ops copilot APIs, investigation console backend | Contact Center |
| 5 Expanded autonomy | 12+ | L2 | L2 actions, learning pipeline, proactive pilots | Risk Committee |

MVP = Phase 0 + Phase 1. Every phase exit = Blueprint VII.2–VII.7 + VII.8 DoD items in scope + the requirement IDs in that phase's gate table.

---

## 3. Phase 0 — Foundation (8–12 weeks)

**Business outcome:** the estate is diagnosable; the Go control plane exists; no customer-facing LLM.

**Business requirements started:** BR-009 (failure intelligence for ops — JourneyTrace viewer), BR-010 (KPI plumbing), BR-012 (phased product coverage — payments taxonomy seed).

### Track A — Estate instrumentation (with platform / payments)

| Step | Work | Req IDs |
|---|---|---|
| A1 | Go middleware + Java interceptor: `x-journey-id` / `traceparent`; RN/React generate them on **all** banking calls | OBS-001, OBS-011 |
| A2 | NGINX `proxy_set_header` + Istio `customTags`; WAF/ALB pass-through tests (X.8) | OBS-001 |
| A3 | Pub/Sub attributes: `journey_id`, `trace_id`, `transaction_id_tok`, `rail`, `lifecycle_stage`, `debit_state`, `error_code`, `event_class` (X.4) | OBS-002, OBS-004, DATA-004, DATA-005, DATA-017 |
| A4 | BQ: day partition, `require_partition_filter`, cluster `transaction_id_tok,journey_id`; 90-day raw expiry; physical billing (X.5 C4–C7) | OBS-007, NFR-017, DATA-009 |
| A5 | **Exit test:** one `journey_id` joins app → NGINX → Istio → Cloud Logging → Pub/Sub → BQ | OBS-001, OBS-003, DATA-017 |

### Track B — Go skeleton (build order)

Each item: scaffold → unit → testcontainers → UAT behind Istio.

| # | Build | What to code | Req IDs |
|---|---|---|---|
| 1 | Shared libs | otel, logging, pubsub, pg, httpx, tokenizer, domain, evidence | OBS-001/005/011, DATA-010, NFR-006 |
| 2 | `audit-service` | Hash-chained append-only writer; durable queue/retry; GCS export stub | AG-009, DATA-015, NFR-008, FR-019, SEC-008, GOV-009 |
| 3 | `policy-service` | Embedded OPA, bundle load, `Check` API | AG-002, SEC-002, FR-009 |
| 4 | `tool-gateway` | Registry table, JSON Schema, deny-by-default, 7 tool classes, kill-switch flag, audit emit; stub tools | AG-002, AG-003, AI-011, SEC-002, SEC-009, API-007, API-008, API-010 |
| 5 | `prompt-registry` | Versioned immutable templates + locale bundles | AI-006, GOV-003, PR-013 |
| 6 | `model-gateway` skeleton | Adapter interface, **PII guard tests (untokenised PII rejected)**, token budgets, unregistered-model reject | NFR-006, AI-001, AI-005, AI-014, SEC-014, CP-002, CP-005 |
| 7 | `evidence-service` | Lazy hydration: Redis T-1 + parameterised BQ + Logging allow-list; **no firehose ingest** | FR-006, OBS-006, DATA-006, SEC-003, X.5 |
| 8 | `transaction-intelligence` skeleton | `resolveJourney` / `getJourneyTrace` via evidence; lazy `txn_trace_map` | FR-003, FR-004, FR-005, DATA-001, DATA-002, DATA-017 |
| 9 | `taxonomy-service` | Rules `(service, api, error_code, state)` → taxonomy_id, template, action class, risk; Redis cache; `tjsa-gap-v1`; seed UPI+IMPS | FR-008, FR-014, FR-015, DATA-013, API-005, API-011, OBS-012 |
| 10 | `kg-ingest` + `code-intelligence-indexer` | OpenAPI/Java → `api_registry` + graph; metadata only, no source text | API-001, API-002, API-004, API-009, API-012, DATA-014, DATA-019 |
| 11 | `ui-knowledge-publisher` + tables | RN/React screens/routes; first procedures (IMPS/NEFT/UPI, add beneficiary) | UI-001, UI-002, UI-003, UI-006, UI-010, UI-012, DATA-020 |
| 12 | `admin-api` skeleton | Read-only JourneyTrace viewer; kill-switch flag store | BR-009, OPS-014 (read), NFR-012, SEC-015, OPS-004 |

### Infra this phase

`agent-platform` namespace, STRICT mTLS, AuthorizationPolicies, Workload Identity (only `intelligence` gets BQ/Logging IAM; `model-gateway` gets none), Istio `/v1/tjsa/**` stubs, NGINX TJSA rate bucket, CI (lint, sqlc, migrate, testcontainers, image signing, **taxonomy CI gate**), Postgres / Redis / Neo4j / Temporal, Cloud KMS tokenizer, Secret Manager.

**NFR / SEC / CP that must be true before Phase 1:** NFR-006, NFR-008, NFR-012 (flag exists), NFR-017, SEC-001 (mesh+session stub), SEC-004, SEC-005, SEC-011, CP-001 (processing inventory), CP-007 (hosting decision E-4 recorded).

### Phase 0 exit gate

| Criterion | Req |
|---|---|
| ≥95% trace reconstruction on sampled txns for 2–3 payment journeys | FR-004, OBS-003, DATA-017 |
| Taxonomy covers 100% of observed error codes on those journeys (4-week sample); CI gate live | FR-008, API-005, API-011, QA-012 |
| PII guard rejects untokenised prompts; tool-gateway denies unknown tools | NFR-006, AI-011, QA-005 |
| Audit hash chain verified; replay of a synthetic investigation | NFR-008, QA-008 |
| LLM hosting / residency decision recorded (E-4) | CP-007, NFR-007 |
| UI registry + procedures exist for first tasks | UI-001, UI-012 |

---

## 4. Phase 1 — Explain-only MVP (12–16 weeks): L0 EXPLAIN + GUIDE + INFORM

**Business outcome:** customer can ask why a payment failed / where money is / how to send IMPS (or add a beneficiary), get a taxonomy-grounded answer, a deep link, or a ticket. **No money-moving or write actions.**

**Business requirements delivered:** BR-001, BR-002, BR-003, BR-004 (deflection via explanation + link + ticket), BR-005, BR-007, BR-011, BR-014; BR-012 started (payments).

### New backend

| Service (deployable) | Code | Req IDs |
|---|---|---|
| `agent-gateway` (`agent-api`) | `POST /v1/tjsa/converse` (SSE), existing JWT validation, session store, per-device limits, channel adapter RN+IB | FR-001, SEC-001, SEC-009, SEC-012, PR-001, PR-002, CP-006 |
| `agent-orchestrator` (`agent-api`) | 15-state investigation SM; intent classifier (II.2.3); planner with tool budget; II.5 / II.5.1 response composer (money position first); one clarifying question; mode switch; `talk to a human` same-turn; FAILED_SAFE | FR-002, FR-019, FR-020, FR-023, FR-024, AG-001, AG-004, AG-005, AG-012, AG-013, AG-014, AI-001, AI-012, PR-003, PR-007, PR-012 |
| `knowledge-service` (`guidance`) | **RAG KB for LLM**: pgvector retrieve → context pack → citations; provenance/version; personal limits via `getCustomerContext` only (not RAG) | AI-003, PR-017, NFR-015, GOV-004, DATA-003, DATA-018, §16.3 |
| `ui-guidance-service` (`guidance`) | `getUIDeepLink`, stale-route, `getTaskGuidance`, `getNextStep`, `checkTaskPrerequisites`, rail/option **rules table** (not LLM), guided-session store | FR-013, FR-021, FR-022, PR-005, PR-006, PR-015, PR-016, PR-018, UI-004, UI-005, UI-007, UI-008, UI-009, UI-011, UI-014, DATA-020 |
| `case-service` (`actions` later; Phase 1 can live in `guidance` or a small extra binary until Phase 2) | Idempotent `createTicket` + SLA + evidence bundle; `escalateToHuman` | FR-010, FR-020, BR-007, BR-014, PR-008, PR-009, PR-010, DATA-008, OBS-010 |

### Extended

- `model-gateway` production: Vertex/Bedrock (E-4), structured JSON validation, fallback, cost metering, A/B **phrasing only** — AI-005, AI-006, AI-007, AI-014, OPS-012.
- `transaction-intelligence` full: rules-first RCA, 13-state financial SM, `getFinancialState`, conflict → escalate — FR-007, FR-008, FR-017, AG-008, AI-002, DATA-012, QA-002.

### Tools registered (all read-only + ticket)

`getCustomerContext`, `resolveJourney`, `getJourneyTrace`, `getFinancialState`, `getEvidence`, `classifyRootCause`, `getExplanation`, `getUIDeepLink`, `getTaskGuidance`, `checkTaskPrerequisites`, `getNextStep`, `getProductFact`, `createTicket`, `escalateToHuman`, `logAgentDecision` (internal).

### Backend HTTP / RPC surface (this phase)

| Endpoint | Deployable | Purpose |
|---|---|---|
| `POST /v1/tjsa/converse` | agent-api | Customer turn (SSE); session JWT |
| `GET /v1/tjsa/conversations/{id}` | agent-api | Resume / PR-009 |
| `POST /v1/internal/tools/{name}` | tool-plane | Orchestrator-only, mTLS |
| `POST /v1/internal/model/complete` | model-gateway | Orchestrator-only |
| `GET /v1/admin/journeys/{id}/trace` | governance | Ops viewer |
| `POST /v1/admin/kill-switch` | governance | NFR-012 |

Internal Connect-Go APIs between deployables match Blueprint V.15.

### Quality before ramp

| Gate | Req |
|---|---|
| Golden set v1 ≥ 200 cases; 15 AI-015 categories seeded (≥30 each before Phase 2 close; Phase 1: payments-heavy) | QA-001, AI-008, AI-015, DATA-016 |
| Red team: injection, PII exfil, out-of-scope, "do the payment for me" | AI-009, SEC-006, FR-024, QA-004, QA-010 |
| Explanation correctness SME-reviewed; jargon detector | FR-011, BR-003, NFR-009 |
| Hallucination / ungrounded claim = release fail | AI-010, QA-004 |
| How-to procedures replayed vs navigation manifest | QA-013, UI-006, UI-012 |
| Load: status P95 ≤ 3 s (no LLM); explain P95 < 4 s; complex → async at 15 s | NFR-001, NFR-002, NFR-004 |
| Chaos: LLM/vector down → deterministic status + ticket | NFR-005, NFR-016, QA-007, AG-013 |
| Kill-switch drill | NFR-012, SEC-015 |
| Same cause → same class/action on RN and IB | NFR-011, QA-011, BR-001 |
| Zero PII in prompt DLP | NFR-006, QA-010 |

### Rollout

Dark → employee → 1% → 100% via existing device-id → bucket routing (X.3). Channel ships chat + CTA + deep-link handler (PR-001, PR-002).

### Phase 1 exit (MVP)

All Phase 0 gates still green, plus: FR-001…015, FR-019…024, PR-001…013, PR-015…018, BR-001…005, BR-007, BR-011, BR-014, NFR-001/002/004/005/006/008/009/011/012/016.

---

## 5. Phase 2 — Safe corrective actions + expansion (12–20 weeks): L1

**Business outcome:** customer can confirm a **whitelisted** reversible action (refresh status, resend confirmation, raise dispute); agent is incident-aware; cards / beneficiaries / mandates covered.

**Business requirements delivered:** BR-006, BR-008; BR-012 next wave.

### New backend

| Service | Code | Req IDs |
|---|---|---|
| `action-workflow` + worker | Action Registry (versioned, risk-tiered, toggle); `listAvailableActions` / `executeAction`; Temporal; idempotency ledger; confirmation token this-turn; step-up via **existing** bank auth; post-action `getFinancialState`; compensation | FR-009, FR-013(b), FR-018, AG-006, AG-007, AG-011, DATA-007, SEC-007, GOV-007, OPS-011, NFR-003 |
| `evidence-ingest` | Filtered subscription `event_class=payment_lifecycle OR success=false`; batched upsert; daily partition drop; HPA + night min (C12); ~600–1,200/s at 12k/s peak | OBS-004, DATA-004, DATA-017, NFR-013, X.5 |
| `journey-intelligence` | Neo4j multi-hop journey/API/dependency | AI-004, DATA-014, FR-005 |
| `incident-service` | `getActiveIncident`; approved generic message; no speculative RCA | FR-016, BR-008, OBS-009, QA-006 |

### Extended

Orchestrator: async investigations (Temporal), incident mode, confirmation dialogs — AG-010, NFR-004, PR-009. Domain content: cards, beneficiaries, mandates — UI-013 wave 2, API-005, FR-021.

### New tools

`listAvailableActions`, `executeAction`, `initiateDispute`, `getActiveIncident`. Refresh status = `executeAction("REFRESH_STATUS")`.

### Extra gates

Idempotency chaos (duplicate submit, worker crash) — AG-007, QA-008; action P95 < 8 s — NFR-003; T-2 freshness with ≤ 60 s backlog at 12k/s; incident drill — FR-016; Action Registry change = maker-checker + audit — GOV-007, OPS-013.

---

## 6. Phase 3 — Deposits, loans, KYC/onboarding, disputes (12–14 weeks)

**No new Go services.** Backend work is configuration, content, and tool registrations.

| Work | Req |
|---|---|
| Taxonomy + rules + success concepts for new domains | FR-008, FR-015, API-005, BR-012 |
| KB + pgvector + freshness owners | AI-003, PR-017, NFR-015, GOV-004 |
| Task procedures + UI routes | FR-021, UI-012, UI-013 |
| Domain read APIs registered on tool-gateway (no gateway rewrite) | API-007, API-010, API-011 |
| Dispute Temporal workflows + case adapter extensions | FR-010, DATA-008 |
| Golden set v3 per domain; red team refresh | QA-001, AI-015 |
| Cost: BQ night aggregates (C6) | OPS-012 |

Exit: each domain passes the same golden-set / red-team / how-to CI gates as payments.

---

## 7. Phase 4 — Contact-centre copilot (8–10 weeks)

**Business:** BR-013, PR-014, FR-012.

| Backend | Req |
|---|---|
| `admin-api`: agent-assist endpoints (same orchestrator, **ops prompt variant**, wider evidence per ops policy); warm-handoff payload | FR-012, PR-010, PR-014, BR-013 |
| Investigation view: customer, conversation, journey, txn, timeline, APIs, trace, errors, RCA, incident, actions, tickets, evidence, confidence, policy, audit; human takeover | OPS-014, BR-009, SEC-013 |
| `agent-gateway` internal channel adapter (existing SSO) | SEC-001, SEC-013 |
| Console RBAC/ABAC, detokenisation reason-capture, access log | SEC-013, DATA-018, CP-004 |

Channel: React ops console (not Go). Exit: handle-time vs control; ops feedback → golden set (OPS-015 start).

---

## 8. Phase 5 — L2 autonomy + learning (12+ weeks)

| Backend | Req |
|---|---|
| Action Registry L2 (execute-then-notify, T1 whitelist); policy bundles | AG-011, GOV-007, BR-006 |
| `learning-pipeline` CronJob (night trough): gap mining, golden candidates, feedback → SME queue; **no auto-train** | AG-015, OPS-015, GOV-011, GOV-012, CP-005, DATA-016 |
| Proactive outbound pilots (Risk Committee) | VII.7.1 — new FR/AG if approved |
| Human review sampling % | GOV-012 |
| Model/knowledge rollback drill | OPS-005 |
| Full DR + backup/restore of PG/audit/Redis | NFR-014, OPS-008, OPS-009 |

---

## 9. Cross-phase backend workstreams (not optional)

These run **every phase**. They are how NFRs and governance stay true while features land.

### 9.1 Security & privacy (SEC, CP)

| Work | When live | IDs |
|---|---|---|
| Channel JWT at gateway; no second login; no tool without session | P1 | FR-001, SEC-001, CP-006 |
| Mesh mTLS, Workload Identity, deny-by-default AuthorizationPolicy | P0 | SEC-004, SEC-011, AG-002 |
| Tokenise before any model or log; DLP on prompts | P0 | NFR-006, DATA-010, CP-002, SEC-014 |
| Customer-scoped queries; adjacent-id tests | P1 | SEC-012 |
| Secrets in Secret Manager only; scanner in CI | P0 | SEC-005, API-008 |
| Prompt/tool injection + poisoned KB red team | P1+ | SEC-006, AI-009 |
| Step-up via existing bank auth (never new OTP) | P2 | SEC-007 |
| Rate limits: converse + stricter on action tools | P1 / P2 | SEC-009, OPS-011 |
| Kill switch: tool / agent / Istio route — drill every phase | P0 flag, P1 drill | SEC-015, NFR-012, OPS-004 |
| SIEM of security decisions | P1 | SEC-008, SEC-010 |
| Processing inventory, residency, vendor register | P0 | CP-001, CP-007, CP-008 |
| No prod training without approval | always | CP-005 |
| Fraud/AML internals → generic "under review" | P1 | CP-011, AI-013 |
| Data-rights hooks to bank privacy workflow | P2+ | CP-004 |
| Disclosure templates | P1 | CP-010 |
| Retention jobs by data class | P0 / P1 | CP-003, DATA-009 |

### 9.2 Observability, SLOs, cost (OBS, NFR, OPS)

| Work | IDs |
|---|---|
| Correlation + structured JSON logs (CI lint) | OBS-001, OBS-011 |
| Lifecycle events on Pub/Sub (schema-governed) | OBS-004 |
| Evidence APIs only — no raw BQ/Logging for LLM | OBS-006 |
| Dashboards: deflection, accuracy vs fallback, action success, P50/P95, token cost, tool fail, guardrail blocks | OBS-008, NFR-010, BR-010 |
| `UNCLASSIFIED_ERROR` spike alert | OBS-012, OPS-003 |
| Evidence snapshots on cases | OBS-010 |
| HPA; size for 12k/s peak, pay average, night batch (C12) | NFR-013, OPS-007, OPS-012 |
| 99.9% + error budget; degrade to ticket | NFR-005, OPS-001, OPS-002 |
| CERT-In 180-day logs in India | NFR-017, OBS-007 |
| Per-turn token budget; deterministic path for simple status; 80% LLM-budget alert | OPS-012, AI-007, NFR-001 |

### 9.3 Governance & quality (GOV, QA)

| Work | IDs |
|---|---|
| AI/model inventory + owners | GOV-001, GOV-002 |
| Versioned prompts, policies, knowledge, actions | GOV-003, GOV-004, GOV-007 |
| Release gates: safety, accuracy, security, performance | GOV-005, GOV-006 |
| Golden regression on **every** prompt/model/taxonomy change | GOV-011, QA-001 |
| Regulatory mapping cadence | GOV-010, NFR-007 |
| Admin maker-checker for journeys, UI nav, mappings, policies, actions, SLAs, models, knowledge | OPS-013 |
| Test pyramid: unit → contract → testcontainers → golden → red team → load → chaos → replay | QA-001…013, V.17 |

### 9.4 Data plane (DATA) — schemas to implement (Blueprint V.5)

| Store | Entities | Phase | IDs |
|---|---|---|---|
| PostgreSQL | `txn_trace_map`, `transaction_event` (daily partitions), `taxonomy_rule`, `api_registry`, `ui_screen`, `task_procedure`, `guided_task_session`, `action_registry`, `action_ledger`, `investigation`, `conversation`, `case_record`, `audit_record` (WORM), `prompt_release`, `policy_bundle` | 0–2 | DATA-001…020 |
| Redis | Session, assembled JourneyTrace (T-1, 2 h), taxonomy hot cache, customer context (TTL = conversation) | 0–1 | DATA-011, DATA-018 |
| BigQuery | Existing envelopes + clustered point-lookup templates; night aggregates | 0 | X.5, OBS-007 |
| Neo4j | 21-entity KG, versioned edges, txn overlay TTL | 0 ingest / 2 queries | DATA-014, DATA-019 |
| GCS | Audit export, knowledge artefacts | 0 | GOV-009 |

---

## 10. Complete backend tool catalogue (implement in Go)

Every tool: name, purpose, inputs, outputs, authz, classification, R/W, audit, rate limit, timeout, idempotency (V.8). Invoked **only** via tool-gateway.

| Tool | Phase | Primary FR | Deployable behind gateway |
|---|---|---|---|
| `getCustomerContext` | 1 | DATA-003, DATA-018 | intelligence |
| `resolveJourney` | 0/1 | FR-003 | intelligence |
| `getJourneyTrace` | 0/1 | FR-004, FR-005 | intelligence |
| `getEvidence` | 0 | FR-006 | intelligence |
| `getFinancialState` | 1 | FR-007, FR-017 | intelligence |
| `classifyRootCause` | 1 | FR-008 | intelligence |
| `getExplanation` | 1 | FR-011, FR-012, FR-015 | intelligence + prompt-registry |
| `getUIDeepLink` | 1 | FR-013, UI-011 | guidance |
| `getTaskGuidance` | 1 | FR-021 | guidance |
| `checkTaskPrerequisites` | 1 | FR-021 | guidance → read APIs |
| `getNextStep` | 1 | FR-022 | guidance |
| `getProductFact` | 1 | PR-017 | guidance |
| `createTicket` | 1 | FR-010 | case-service |
| `escalateToHuman` | 1 | FR-020 | case-service |
| `logAgentDecision` | 0/1 | FR-019 | audit-writer |
| `getActiveIncident` | 2 | FR-016 | incident-service |
| `listAvailableActions` | 2 | FR-009, AG-011 | actions |
| `executeAction` | 2 | FR-018, AG-006/007/011 | actions |
| `initiateDispute` | 2/3 | FR-010 | actions |

---

## 11. Master requirements → phase → deployable (complete register)

Phase = first phase the requirement must be **accepted**. Later phases keep it green.

### 11.1 Business (BR)

| ID | Requirement | Phase | Backend owner |
|---|---|---|---|
| BR-001 | Unified MB + IB, one engine | 1 | agent-api |
| BR-002 | Transaction-specific, authoritative | 1 | intelligence |
| BR-003 | Human-readable, no internals | 1 | orchestrator output policy |
| BR-004 | Reduce avoidable escalation | 1 | agent-api + guidance + case |
| BR-005 | Channel-specific next steps | 1 | guidance |
| BR-006 | Self-service low-risk actions | 2 | actions |
| BR-007 | Service request + evidence | 1 | case-service |
| BR-008 | Customer vs incident | 2 | incident-service |
| BR-009 | Structured failure intel for ops | 0/4 | governance |
| BR-010 | Measurable KPIs | 1 | OBS dashboards |
| BR-011 | Explain successes | 1 | taxonomy + intelligence |
| BR-012 | Every product line, phased | 1→5 | content + taxonomy |
| BR-013 | Ops/L1 copilot | 4 | admin-api + agent-api |
| BR-014 | Immediate human path | 1 | case-service + orchestrator |

### 11.2 Product (PR)

| ID | Phase | Backend owner |
|---|---|---|
| PR-001 | 1 | agent-gateway API; **channel** RN sheet |
| PR-002 | 1 | same API; **channel** React |
| PR-003 | 1 | orchestrator |
| PR-004 | 1 | timeline allow-list in intelligence |
| PR-005 | 1 | ui-guidance (RN version) |
| PR-006 | 1 | ui-guidance (web version) |
| PR-007 | 1 | response composer |
| PR-008 | 1 | case-service |
| PR-009 | 1 | agent-api + case polling |
| PR-010 | 1 | escalate payload |
| PR-011 | 1 | = BR-011 |
| PR-012 | 1 | conversation memory |
| PR-013 | 1 | prompt-registry locales |
| PR-014 | 4 | ops prompt + console API |
| PR-015 | 1 | Task Guidance Engine |
| PR-016 | 1 | getNextStep + completion |
| PR-017 | 1 | knowledge-service |
| PR-018 | 1 | rail/option **rules** |

### 11.3 Functional (FR)

| ID | Phase | Backend owner |
|---|---|---|
| FR-001 | 1 | agent-gateway |
| FR-002 | 1 | orchestrator |
| FR-003 | 0/1 | transaction-intelligence |
| FR-004 | 0/1 | transaction-intelligence |
| FR-005 | 0/2 | transaction- + journey-intelligence |
| FR-006 | 0 | evidence-service |
| FR-007 | 1 | transaction-intelligence |
| FR-008 | 0/1 | taxonomy-service |
| FR-009 | 2 | policy + action-workflow |
| FR-010 | 1 | case-service |
| FR-011 | 1 | getExplanation + composer |
| FR-012 | 4 | ops variant |
| FR-013 | 1 (a,c) / 2 (b) | guidance / actions / case |
| FR-014 | 0/1 | taxonomy + case |
| FR-015 | 1 | taxonomy success concepts |
| FR-016 | 2 | incident-service |
| FR-017 | 1 | financial state machine |
| FR-018 | 2 | action-workflow |
| FR-019 | 0/1 | audit-service |
| FR-020 | 1 | orchestrator + case |
| FR-021 | 1 | ui-guidance |
| FR-022 | 1 | ui-guidance |
| FR-023 | 1 | orchestrator |
| FR-024 | 1 | tool-gateway + orchestrator |

### 11.4 Non-functional (NFR)

| ID | Phase accepted | How implemented |
|---|---|---|
| NFR-001 | 1 | Deterministic status path; load test at peak |
| NFR-002 | 1 | Explain path load test < 4 s P95 |
| NFR-003 | 2 | Action + verify < 8 s P95 |
| NFR-004 | 1 | 15 s budget → Temporal + ticket |
| NFR-005 | 1 | 99.9%, degrade to ticket; chaos |
| NFR-006 | 0 | Tokenizer + model-gateway DLP |
| NFR-007 | 0/1 | Compliance sign-off + GOV-010 |
| NFR-008 | 0 | Audit writer + replay test |
| NFR-009 | 1 | taxonomy_id on every explanation |
| NFR-010 | 1 | Grafana / AI Log Analyzer |
| NFR-011 | 1 | Golden same-cause / both channels |
| NFR-012 | 0/1 | Flag + Istio runbook + drill |
| NFR-013 | 1/2 | HPA; 12k/s design, filter T-2 |
| NFR-014 | 5 (drill P2+) | RPO/RTO per store |
| NFR-015 | 1 | Owner + expiry on KB/UI |
| NFR-016 | 1 | LLM/vector down → status + ticket |
| NFR-017 | 0 | 180-day logs in India |

### 11.5 AI

| ID | Phase | Owner |
|---|---|---|
| AI-001 | 1 | model-gateway + orchestrator |
| AI-002 | 1 | taxonomy + state machine |
| AI-003 | 1 | knowledge-service |
| AI-004 | 2 | journey-intelligence |
| AI-005 | 0/1 | model-gateway JSON Schema |
| AI-006 | 0 | prompt-registry + audit |
| AI-007 | 1 | routing table |
| AI-008 | 1 | eval harness |
| AI-009 | 1 | gateway + red team |
| AI-010 | 1 | grounding evaluator |
| AI-011 | 0 | tool-gateway |
| AI-012 | 1 | conversation store |
| AI-013 | 1 | taxonomy fraud templates |
| AI-014 | 0/1 | model registry |
| AI-015 | 1→2 | golden-set coverage |

### 11.6 Agent (AG)

| ID | Phase | Owner |
|---|---|---|
| AG-001 | 1 | orchestrator SM |
| AG-002 | 0 | tool-gateway + mesh |
| AG-003 | 0 | tool classes |
| AG-004 | 1 | evidence thresholds |
| AG-005 | 1 | composer |
| AG-006 | 2 | action-workflow |
| AG-007 | 2 | idempotency ledger |
| AG-008 | 1 | financial SM |
| AG-009 | 0 | audit-service |
| AG-010 | 2 | Temporal |
| AG-011 | 2 | Action Registry |
| AG-012 | 1 | orchestrator |
| AG-013 | 1 | degrade path |
| AG-014 | 1 | SM mapping |
| AG-015 | 5 | learning-pipeline |

### 11.7 Data

| ID | Phase | Store / job |
|---|---|---|
| DATA-001 | 0 | PG Transaction |
| DATA-002 | 0 | PG Journey |
| DATA-003 | 1 | customer context schema |
| DATA-004 | 0 | events + Pub/Sub schema |
| DATA-005 | 0 | API execution (envelope) |
| DATA-006 | 0 | Evidence Objects |
| DATA-007 | 2 | action_ledger |
| DATA-008 | 1 | case_record |
| DATA-009 | 0 | retention jobs |
| DATA-010 | 0 | tokenizer |
| DATA-011 | 1 | investigation + Redis TTL |
| DATA-012 | 1 | 13-state model |
| DATA-013 | 0 | taxonomy_rule |
| DATA-014 | 0/2 | Neo4j |
| DATA-015 | 0 | audit WORM |
| DATA-016 | 1 | golden dataset |
| DATA-017 | 0 | txn_trace_map |
| DATA-018 | 1 | customer-context fields |
| DATA-019 | 0 | kg-ingest lifecycle |
| DATA-020 | 1 | task_procedure + guided_session |

### 11.8 API intelligence

| ID | Phase | Owner |
|---|---|---|
| API-001 | 0 | kg-ingest |
| API-002 | 0 | SME + registry |
| API-003 | 0 | UI↔API map |
| API-004 | 0 | kg-ingest |
| API-005 | 0 | taxonomy |
| API-006 | 0 | envelope + evidence |
| API-007 | 0 | tool-gateway metadata tools |
| API-008 | 0 | field policy |
| API-009 | 0 | registry |
| API-010 | 0 | contract tests |
| API-011 | 0 | CI on Go/Java repos |
| API-012 | 2 | code-intelligence-indexer |

### 11.9 UI intelligence

| ID | Phase | Owner |
|---|---|---|
| UI-001 | 0 | publisher + PG |
| UI-002 | 0 | publisher + PG |
| UI-003 | 0 | map |
| UI-004 | 1 | getUIDeepLink |
| UI-005 | 1 | fallback routes |
| UI-006 | 0 | CI vs release |
| UI-007 | 1 | a11y labels |
| UI-008 | 1 | locale variants |
| UI-009 | 0 | customer-safe labels |
| UI-010 | 0 | admin workflow |
| UI-011 | 1 | deep-link resolver |
| UI-012 | 0/1 | task_procedure |
| UI-013 | 1→3 | coverage waves |
| UI-014 | 1 | locale/a11y procedures |

### 11.10 Observability

| ID | Phase | Owner |
|---|---|---|
| OBS-001 | 0 | otel + edge |
| OBS-002 | 0 | lifecycle spans |
| OBS-003 | 0 | join test |
| OBS-004 | 0/2 | Pub/Sub + ingest |
| OBS-005 | 0 | redaction |
| OBS-006 | 0 | evidence-service |
| OBS-007 | 0 | retention |
| OBS-008 | 1 | dashboards |
| OBS-009 | 2 | incident-service |
| OBS-010 | 1 | case snapshots |
| OBS-011 | 0 | log schema lint |
| OBS-012 | 0/1 | alert |

### 11.11 Security

| ID | Phase | Owner |
|---|---|---|
| SEC-001 | 1 | agent-gateway |
| SEC-002 | 0 | tool-plane |
| SEC-003 | 0 | evidence allow-list |
| SEC-004 | 0 | mTLS + KMS |
| SEC-005 | 0 | Secret Manager |
| SEC-006 | 1 | red team |
| SEC-007 | 2 | existing step-up |
| SEC-008 | 1 | audit → SIEM |
| SEC-009 | 1 | NGINX + gateway |
| SEC-010 | 1 | IR runbook |
| SEC-011 | 0 | Workload Identity |
| SEC-012 | 1 | customer scope |
| SEC-013 | 4 | console RBAC |
| SEC-014 | 0 | DLP + egress |
| SEC-015 | 0/1 | kill switch |

### 11.12 Governance

| ID | Phase | Owner |
|---|---|---|
| GOV-001 | 0 | inventory |
| GOV-002 | 0 | RACI |
| GOV-003 | 0 | prompt-registry |
| GOV-004 | 1 | knowledge versions |
| GOV-005 | 1 | release gates |
| GOV-006 | 1 | red team |
| GOV-007 | 2 | Action Registry audit |
| GOV-008 | 1 | AI incident taxonomy |
| GOV-009 | 0 | audit export |
| GOV-010 | 0 | regulatory cadence |
| GOV-011 | 1 | golden on every change |
| GOV-012 | 5 | sampling |

### 11.13 Quality

| ID | Phase | Suite |
|---|---|---|
| QA-001 | 1 | golden datasets |
| QA-002 | 1 | state reconstruction |
| QA-003 | 1 | UI nav by version |
| QA-004 | 1 | hallucination |
| QA-005 | 0 | tool authz |
| QA-006 | 2 | incident-mode |
| QA-007 | 1 | degraded-mode |
| QA-008 | 0 | replay / audit |
| QA-009 | 1 | latency / concurrency |
| QA-010 | 1 | leakage / privacy |
| QA-011 | 1 | consistency |
| QA-012 | 0 | taxonomy CI |
| QA-013 | 1 | how-to procedures |

### 11.14 Operations

| ID | Phase | Owner |
|---|---|---|
| OPS-001 | 1 | runbooks |
| OPS-002 | 1 | health dashboards |
| OPS-003 | 1 | unsafe-behaviour alerts |
| OPS-004 | 0/1 | write kill switch |
| OPS-005 | 1/5 | model/knowledge rollback |
| OPS-006 | 2 | sanitised replay |
| OPS-007 | 1 | quotas |
| OPS-008 | 2+ | DR test |
| OPS-009 | 2+ | backup/restore |
| OPS-010 | 1 | freshness monitor |
| OPS-011 | 2 | action-tool limits |
| OPS-012 | 1 | cost mgmt |
| OPS-013 | 0/2 | admin maker-checker |
| OPS-014 | 0/4 | ops console API |
| OPS-015 | 5 | learning loop |

### 11.15 Compliance & privacy (CP)

| ID | Phase | Owner |
|---|---|---|
| CP-001 | 0 | inventory |
| CP-002 | 0 | minimise + tokenise |
| CP-003 | 0 | retention jobs |
| CP-004 | 2+ | privacy workflow |
| CP-005 | 0 | no uncontrolled training |
| CP-006 | 1 | consent scope on session |
| CP-007 | 0 | India residency decision |
| CP-008 | 0 | vendor register |
| CP-009 | 1 | audit export |
| CP-010 | 1 | limitation disclosures |
| CP-011 | 1 | fraud/AML generic text |

**Coverage check:** BR 14 + PR 18 + FR 24 + NFR 17 + AI 15 + AG 15 + DATA 20 + API 12 + UI 14 + OBS 12 + SEC 15 + GOV 12 + QA 13 + OPS 15 + CP 11 = **227**. All appear once in §11.

---

## 12. Cross-phase Definition of Done (backend production)

Maps Blueprint VII.8 to this plan:

| # | DoD | Phase first true | IDs |
|---|---|---|---|
| 1 | Journeys mapped to versioned UI + API metadata | 0/1 | UI-*, API-001…003 |
| 2 | Correlation IDs end-to-end | 0 | OBS-001 |
| 3 | Authority matrix signed | 1 | FR-007, DATA-012 |
| 4 | State machine + conflict rules tested per rail | 1 | FR-017, AG-008, QA-002 |
| 5 | Evidence service + PII controls | 0 | FR-006, NFR-006 |
| 6 | Policy/action outside LLM | 0/2 | AG-002, AG-011 |
| 7 | Golden thresholds | 1 | QA-001, AI-008 |
| 8 | Injection / tool-abuse red team | 1 | AI-009, QA-005 |
| 9 | Kill switch drilled | 1 | NFR-012 |
| 10 | Case integration with evidence | 1 | FR-010, OBS-010 |
| 11 | DR + degraded modes | 1 / 2+ | NFR-014, NFR-016 |
| 12 | Inventories owned and versioned | 0 | GOV-001…004 |
| 13 | Compliance / Security / Risk / Legal sign-off | 1 | NFR-007 |
| 14 | No new API without taxonomy entry | 0 | API-011, QA-012 |

---

## 13. Engineering rules (binding)

1. **Repo:** one Go monorepo; `internal/domain` has zero external imports; sqlc + golang-migrate; Buf for Connect/proto.
2. **Composition:** logical packages stay split; deployables follow §1a; B1–B3 never violated.
3. **Tool path:** orchestrator → tool-gateway → domain. No LLM shortcuts. `model-gateway` has no data-store IAM.
4. **Version everything** that is not application code: schema, prompt, policy, taxonomy, procedure, action. Independent of binaries.
5. **Diurnal (C12):** HPA night minima; batch in the trough; bounded subscriber backlog at 12k/s.
6. **Kill switches** tested every phase (tool, agent, Istio).
7. **CI blocks** on QA-005, QA-010, QA-012, and golden/red-team gates once those suites exist.

## 14. Team shape (Blueprint VII.9)

| Role | Phase 0–1 | Phase 2+ |
|---|---|---|
| Go platform engineers | 4–6 (scale to 6–8) | 6–8 |
| Data / BQ engineer | 1–2 | 1 |
| Java instrumentation (existing services) | 2 | 0–1 |
| App engineers (RN/React, part-time) | 2 | 1–2 |
| SME / taxonomy | 2 | 2–3 per domain |
| AI / prompt / eval | 1–2 | 2 |
| Security | 1 | 1 |
| Risk / Compliance | 1 | 1 |
| SRE | 1–2 | 2 |

---

## 15. Kick-off blockers (owners before Phase 0)

Blueprint I.11 + Volume X E-1…E-9, especially: LLM hosting / residency (E-4, CP-007); Pub/Sub **attribute** producer change (E-7, OBS-004); BQ partition/clustering as-is vs C4–C7 (E-8); dedicated host vs path (E-1); envelope vs parallel topic (E-2).

**These do not block starting Phase 0 Go code.** They block *production* LLM, *EXPLAIN* answers, and *ticket* adapters. Use the defaults in §16 until owners confirm.

---

## 16. Backend-only start — UI is out of this workstream

**Decision (binding for this delivery):** no React / React Native / ops-console UI will be built by the backend team. Frontend developers own all channel surfaces (PR-001, PR-002 chat sheet, “Explain this txn” CTA, deep-link handler, later ops console). Backend owns every Go service, API, schema, tool, and gate in this plan.

### 16.1 Can coding start now?

| Question | Answer |
|---|---|
| Start **Phase 0** backend in Go this week, with no UI? | **Yes.** Phase 0 has no customer screen. Prove it with `curl` / Connect clients / the admin JourneyTrace API. |
| Start coding the **entire** backend (Phases 0–5) in one go? | **No.** Phase 1+ needs a frozen converse contract, taxonomy seed, and (for production LLM) hosting sign-off. Phase 2 needs Action Registry legal approval (I.11 #9). Phase 3–5 are content/config on the Phase 1–2 platform. |
| Blocked on frontend? | **No.** FE can start later against the OpenAPI we publish in Phase 0 week 2. |
| Blocked on open questions? | **Not for skeleton code.** Use the defaults below. Blocked only for *production* model calls, live tickets, and live actions. |

### 16.2 Code in this order (no UI)

**Week 1–2 (start immediately)**

1. Repo: `tjsa/` monorepo, `cmd/` + `internal/` as §1, golang-migrate, sqlc, Buf, CI (lint, test, image).
2. Shared libs: `internal/platform/{otel,logging,pubsub,pg,httpx,tokenizer}`, `internal/domain`.
3. Deployables (clubbed, §1a): `audit-writer`, `tool-plane` (gateway + OPA + prompt-registry), `governance` (admin-api skeleton), `intelligence` (evidence lazy + taxonomy + `resolveJourney`/`getJourneyTrace` stubs), `model-gateway` skeleton with PII-reject tests (sandbox LLM or recorded fixtures — **no prod provider required**).
4. Istio `/v1/tjsa/**` stubs + health/readiness in UAT.
5. **Publish `openapi/tjsa-converse.v1.yaml`** (even if the handler returns 501) so FE can mock. This is the only backend→FE deliverable they need to start.

**Still Phase 0, needs bank counterparts (not FE)**

- Correlation headers through NGINX/Istio/services (Track A — platform/payments).
- Pub/Sub attributes + BQ clustering (Track A).
- Taxonomy seed workshop (UPI+IMPS SMEs).
- `ui-knowledge-publisher` reads the **existing app repo** (routes/screens). Backend stores the registry; FE does not build a new UI for this.

**Do not code yet**

| Work | Why wait |
|---|---|
| Production `model-gateway` provider | I.11 #1 / E-4 residency |
| `case-service` live adapter | I.11 #2 ticketing SoR — implement the **port**; adapter after decision |
| `executeAction` / Temporal money-adjacent writes | I.11 #9 Action Registry approval — Phase 2 |
| `agent-api` converse loop against a real LLM | After Phase 0 exit test (journey_id join) + golden fixtures |
| Any RN/React/ops HTML | Explicitly out of scope |

**Safe defaults until open questions close**

| Open question | Code default |
|---|---|
| I.11 #1 / E-4 LLM hosting | Adapter interface + fixture/sandbox; swap provider without rewrite |
| I.11 #2 Ticketing | `CasePort` interface; no-op/in-memory adapter |
| I.11 #3 Voice | Out of v1 — no code |
| I.11 #4 Locales | `en-IN` templates only; registry is locale-keyed |
| I.11 #5 Volume rails | UPI + IMPS first |
| E-1 Host vs path | Path prefix `/v1/tjsa/` on existing app host |
| E-7 Pub/Sub attributes | Schema + subscriber filter ready; producers can lag — lazy BQ path still works |

### 16.3 Contract to hand the frontend team (when they start)

Backend freezes and versions this; FE does not invent it. **CQRS paths (Blueprint §14.1.3 / X.11) are the contract of record.**

| Item | Notes |
|---|---|
| **Query** `POST /v1/tjsa/q/converse` | SSE; existing channel JWT; body: `{conversation_id?, message, locale, channel, app_version, transaction_id?}` — EXPLAIN/GUIDE/INFORM |
| **Query** `GET /v1/tjsa/q/conversations/{id}` | Resume |
| **Query** `GET /v1/tjsa/q/journeys/{ref}` | Optional status/timeline (no LLM) |
| **Command** `POST /v1/tjsa/c/tickets` | Idempotency-Key; create ticket |
| **Command** `POST /v1/tjsa/c/actions` | Idempotency-Key; confirmation token; execute whitelisted action |
| **Command** `POST /v1/tjsa/c/escalate` | Human handoff |
| Legacy alias | `POST /v1/tjsa/converse` → Query until FE cut-over |
| Response envelope | II.5 / II.5.1: `mode`, `money_position`, `explanation`, `next_step` (`deep_link` \| `command_offer` \| `ticket` \| `none`), `taxonomy_id`, `confidence`, `citations[]` (RAG `source_id`+version), `ticket_ref?` |
| Deep links | `bankapp://…` / web routes from `getUIDeepLink` — FE **handles**; backend **resolves** |
| Errors | `FAILED_SAFE`, `CLARIFY` (2–3 candidates), `ESCALATE` |
| Out of FE scope | Tools, BQ, taxonomy, RAG corpus, audit, kill switch, model |

FE can develop the chat sheet against a stub/mock of this spec in parallel with Phase 1 backend. **No FE work is required to start or finish Phase 0.**

---

## 17. CQRS + RAG (both available — binding)

Aligned with Blueprint v2.4 §§14.1.3, 16.3, X.11. **Both are in scope for this backend workstream.** UI remains FE-owned.

### 17.1 CQRS package / API layout

```
cmd/agent-gateway/          # routes /q/** and /c/** to orchestrator modes
internal/query/             # converse query, journey read, response composer
internal/command/           # tickets, actions, escalate, idempotency
internal/readmodel/         # JourneyReadModel, TransactionReadModel, TaskSessionReadModel
internal/projection/        # Pub/Sub → PG (evidence-ingest)
internal/knowledge/         # RAG retrieve, pack, cite (knowledge-service)
```

| Plane | Go packages | Deployable | Phase live |
|---|---|---|---|
| Query | `internal/query`, `internal/readmodel`, `internal/knowledge`, intelligence tools | `agent-api` + `intelligence` + `guidance` + `model-gateway` | P1 (skeleton P0) |
| Command | `internal/command`, action-workflow, case-service | `agent-api` + `actions` + `tool-plane` | P1 tickets; P2 actions |
| Shared | tool-gateway classes (query vs command), audit, policy | `tool-plane`, `audit-writer` | P0 |

**Hard rules in code**

1. Query handlers register only query tool classes; command handlers only command tools — enforced in `tool-gateway` (AG-003).
2. `model-gateway` is called from Query path; Command path may call LLM only to *draft* confirmation copy, never to authorize a write.
3. After every successful command, call Query `getFinancialState` (or ticket get) before telling the customer it worked (FR-018).

### 17.2 RAG knowledge base for the LLM (implement)

| Work | Phase | Detail |
|---|---|---|
| PG schema `kb_document`, `kb_chunk` (embedding vector, source_id, version, authority, locale, domain, valid_from/to) | 0 | DATA / AI-003 |
| Ingest job: approve → PII scan → chunk → embed → upsert | 0 | Document pipeline; night trough for reindex |
| `knowledge-service` APIs: `Retrieve(query, filters) → ContextPack` | 0–1 | Used only by Query tools |
| Tool `getProductFact` + internal retrieve for EXPLAIN/INFORM | 1 | PR-017 |
| Prompt templates require pack grounding; grounding evaluator in CI | 1 | AI-010, QA-004 |
| Seed corpus: product limits/charges/cut-offs, IMPS/UPI how-to narrative, explanation templates | 1 | SME + Compliance |
| Expand corpus per domain wave | 3 | BR-012 |

**Not in RAG:** balances, txn state, RCA codes, action permissions — those stay taxonomy/state/Action Registry.

**Context pack shape (to Model Gateway):**

```json
{
  "chunks": [
    {"source_id": "prod.imps.limits", "version": "2026.09", "authority": "product", "text": "..."}
  ],
  "token_budget_used": 1200
}
```

### 17.3 Phase wiring

| Phase | CQRS | RAG |
|---|---|---|
| 0 | `/q` and `/c` route stubs; read-model tables; projection interface | Schema + ingest + empty/seed index; PII-reject tests |
| 1 | Query live (EXPLAIN/GUIDE/INFORM); Command: tickets + escalate | Live INFORM + EXPLAIN enrichment; citations in audit |
| 2 | Command: Action Registry + Temporal | Corpus growth; ops-only runbook collection |
| 3+ | Same planes; more domain read models | Domain KB packs |

### 17.4 Requirements touched

CQRS: BR-001, BR-006, FR-001–024 (split by plane), AG-002/003/011, SEC-002/009, NFR-001–004.  
RAG: AI-003, AI-010, PR-017, GOV-004, NFR-015, DATA-013 (KB side), OPS-010, CP-002/005.

---

*Plan v2.2 — CQRS + RAG both available; backend-only start; UI out of this workstream. Re-baseline weeks after Phase 0 discovery (Blueprint VII.1).*
