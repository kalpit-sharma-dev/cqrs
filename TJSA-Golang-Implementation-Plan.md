# TJSA — Golang Phase-wise Implementation Plan

**Companion to:** `Autonomous-Banking-AI-Agent-BRD-PRD-HLD-LLD-Blueprint.md` v2.3 (the Blueprint).
This is the **engineering execution plan**: what to build, in what order, in which phase, with which Go services. Design authority stays with the Blueprint — section references are given throughout. Estate decisions from Blueprint Volume X are binding here (GCP, GKE + ASM/Istio, Pub/Sub, BigQuery, Cloud Logging, existing edge and auth, 15+ crore events/12 daytime hours, 12k events/s peak).

---

## 1. Complete service inventory

Everything is Go except one Node build tool. One monorepo (`tjsa/`), hexagonal layout per Blueprint V.2 (`cmd/<service>` + shared `internal/`), one binary per service, all deployed to the existing GKE clusters in namespace `agent-platform` behind Istio (Blueprint X.2–X.3).

| # | Service | Binary | Purpose (Blueprint ref) | Build phase | Key dependencies |
|---|---|---|---|---|---|
| 1 | `agent-gateway` | `cmd/agent-gateway` | External conversation API, OIDC session validation, SSE, rate limits (V.1) | 1 | Existing IDP, Redis, orchestrator |
| 2 | `agent-orchestrator` | `cmd/agent-orchestrator` | Investigation state machine, planner, tool sequencing, response composer (§25, V.10) | 1 | tool-gateway, model-gateway, policy-service, Redis, PG |
| 3 | `tool-gateway` | `cmd/tool-gateway` | Tool registry, JSON-schema validation, authz, kill switch, audit fan-out (V.8) | 0 | policy-service, domain services |
| 4 | `model-gateway` | `cmd/model-gateway` | LLM provider adapters, PII guard, token budgets, structured-output validation (V.13) | 0 (skeleton) → 1 (full) | prompt-registry, KMS, LLM provider |
| 5 | `prompt-registry` | `cmd/prompt-registry` | Versioned prompt templates, locale bundles (V.14) | 0 | PG |
| 6 | `policy-service` | `cmd/policy-service` | Embedded OPA, policy bundles, `Check` API (§30, V.9) | 0 | PG |
| 7 | `transaction-intelligence` | `cmd/transaction-intelligence` | `resolveJourney`, `getJourneyTrace`, `getFinancialState`, RCA + lifecycle state engine (§20, §24) | 0 (trace stitch) → 1 (full) | evidence-service, PG, core status APIs |
| 8 | `evidence-service` | `cmd/evidence-service` | Curated evidence over **BigQuery + Cloud Logging + Redis cache**, lazy hydration (X.5) | 0 | BQ, Cloud Logging, Redis, PG |
| 9 | `taxonomy-service` | `cmd/taxonomy-service` | Error-taxonomy rules lookup, templates, gap logging (§23) | 0 | PG, Redis |
| 10 | `knowledge-service` | `cmd/knowledge-service` | RAG retrieval with source/version filters (§16, §37) | 1 | PG + pgvector, Neo4j |
| 11 | `ui-guidance-service` | `cmd/ui-guidance-service` | Deep-link resolver, Task Guidance Engine, prerequisite checks, rail advisor (§19) | 1 | PG, Redis, Pub/Sub, tool-gateway |
| 12 | `case-service` | `cmd/case-service` | Idempotent ticket create, adapters, status polling (§28) | 1 | PG, ticketing system |
| 13 | `action-workflow` | `cmd/action-workflow` + `cmd/action-worker` | Action Registry, `executeAction`, Temporal workflows, idempotency ledger (§27, V.12) | 2 | Temporal, PG, step-up auth, bank APIs |
| 14 | `journey-intelligence` | `cmd/journey-intelligence` | Journey model, graph queries (§18) | 2 | Neo4j, PG |
| 15 | `incident-service` | `cmd/incident-service` | Incident registry, impact correlation (§39) | 2 | PG, IMS |
| 16 | `audit-service` | `cmd/audit-service` | Append-only hash-chained audit, export (§22, V.7) | 0 | PG (WORM), GCS |
| 17 | `admin-api` | `cmd/admin-api` | Versioned config admin, kill switches, consoles backend (§53–54) | 0 (skeleton) → 4 (copilot) | PG, audit |
| 18 | `kg-ingest` | `cmd/kg-ingest` | OpenAPI/Java contracts/code index → registries, graph, KB (§37.4) | 0 (batch) | GCS, PG, Neo4j |
| 19 | `code-intelligence-indexer` | `cmd/code-intelligence-indexer` | Go/Java AST + OpenAPI + UI nav → Code Intelligence Index (§38) | 0 (batch) | Repos read-only |
| 20 | `evidence-ingest` | `cmd/evidence-ingest` | **Filtered** Pub/Sub subscriber → T-2 hot store, batched idempotent upserts (X.5.1) | 2 (deferred by design — lazy hydration first) | Pub/Sub, PG |
| 21 | `ui-knowledge-publisher` | Node CLI (only non-Go artefact) | Extracts RN/React routes/screens into UI registry (§19) | 0 | App repos, PG |

**Shared internal libraries (built in Phase 0, before any service):**

| Package | Contents |
|---|---|
| `internal/platform/otel` | Correlation middleware: `x-journey-id`/`traceparent` extract→log→propagate (X.8) |
| `internal/platform/logging` | `slog` JSON schema per OBS-011, PII-safe field types |
| `internal/platform/pubsub` | Publisher/subscriber wrappers, ordering keys, idempotent-upsert helpers keyed by `event_id` |
| `internal/platform/pg` | pgx pool, sqlc conventions, golang-migrate harness, partition-drop jobs |
| `internal/platform/httpx` / `rpc` | chi + Connect-Go scaffolding, health/readiness, auth middleware |
| `internal/domain` | Zero-dependency domain types: Journey, TransactionEvent, EvidenceObject, TaxonomyEntry, Action, Investigation |
| `internal/evidence` | BQ template queries (parameterised only, `maximum_bytes_billed`), Logging reader, allow-list, tokenizer client |

Standard library set (Blueprint V.2): `chi`, `connectrpc.com/connect`, `pgx/v5` + `sqlc`, `go-redis/v9`, `cloud.google.com/go/pubsub` + `bigquery` + `logging`, `neo4j-go-driver/v5`, `go.temporal.io/sdk`, `opa/rego`, `santhosh-tekuri/jsonschema/v6`, `anthropic-sdk-go` (Vertex/Bedrock adapters), `testify`, `testcontainers-go`.

---

## 2. Phase plan at a glance

| Phase | Weeks (indicative, VII.1) | Autonomy | Services **new** | Services **extended** | Gate owner |
|---|---|---|---|---|---|
| 0 Foundation | 8–12 | — | tool-gateway, policy-service, prompt-registry, model-gateway (skeleton), audit-service, admin-api (skeleton), evidence-service, transaction-intelligence (skeleton), taxonomy-service, kg-ingest, code-intelligence-indexer, ui-knowledge-publisher | — | CTO / EA |
| 1 Explain-only MVP | 12–16 | L0 | agent-gateway, agent-orchestrator, knowledge-service, ui-guidance-service, case-service | model-gateway (full), transaction-intelligence (full RCA) | Digital Banking + Risk |
| 2 Safe actions | 12–20 | L1 | action-workflow (+worker), journey-intelligence, incident-service, evidence-ingest (T-2) | orchestrator (async), evidence-service (T-2 tier) | Risk + Security |
| 3 Domain expansion | 12–14 | L1 | — (content/config, not services) | taxonomy, knowledge, ui-guidance, action-workflow, case-service | Product + Compliance |
| 4 Contact-centre copilot | 8–10 | L1 | ops console UI (React, non-Go) | admin-api, agent-gateway (agent-assist mode) | Contact Center |
| 5 Expanded autonomy | 12+ | L2 | learning-pipeline batch jobs | action-workflow, policy-service, admin-api | Risk Committee |

MVP = Phase 0 + Phase 1. Every phase ends with the Blueprint's exit criteria (VII.2–VII.7) plus the cross-phase DoD (VII.8).

---

## 3. Phase 0 — Foundation (8–12 weeks): make the estate diagnosable, stand up the platform skeleton

No LLM in production this phase. Two parallel tracks.

### Track A — Estate instrumentation (with platform/payments teams)

| Step | Work | Output |
|---|---|---|
| A1 | Build `internal/platform/otel` middleware (Go) + Java interceptor; app teams add `x-journey-id`/`traceparent` generation in the RN/React network layer | Correlation ids at every hop |
| A2 | NGINX `proxy_set_header` + Istio `tracing.customTags` config; verify ALB/WAF pass-through with synthetic requests (X.8 checklist) | Headers survive the full edge chain |
| A3 | Pub/Sub envelope enrichment: add `journey_id`, `trace_id`, `transaction_id_tok`, `rail`, `lifecycle_stage`, `debit_state`, `error_code` **as message attributes** incl. `event_class` (X.4) | Filterable, joinable stream |
| A4 | BigQuery telemetry tables: partition by ingestion day, `require_partition_filter`, cluster by `transaction_id_tok`,`journey_id`; 90-day raw expiry; physical storage billing (X.5.3 C4–C7) | Cheap point lookups |
| A5 | **Phase 0 exit test:** one `journey_id` joins app → NGINX log → Istio telemetry → Cloud Logging → Pub/Sub envelope → BQ row (X.8) | Go/no-go for EXPLAIN mode |

### Track B — Go platform skeleton (TJSA team)

Build order (each service: scaffold → unit tests → testcontainers integration tests → deploy to UAT behind Istio):

1. **Shared libraries** (`internal/platform/*`, `internal/domain`) — everything depends on them.
2. **`audit-service`** — hash-chained append-only writes; every later service emits to it from day one.
3. **`policy-service`** — embedded OPA, bundle loading, `Check` API.
4. **`tool-gateway`** — tool registry table, JSON-schema validation, deny-by-default authz, kill switch flag, audit emit. Register stub tools.
5. **`prompt-registry`** — versioned templates CRUD, immutable releases.
6. **`model-gateway` (skeleton)** — provider adapter interface, **PII guard (regex + tokenizer client) with tests proving untokenised PII is rejected**, token budget counters; wire to a sandbox LLM only.
7. **`evidence-service`** — lazy-hydration mode (X.5.1): Redis T-1 cache + parameterised BQ template queries + Cloud Logging reader + attribute allow-list. No ingest pipeline.
8. **`transaction-intelligence` (skeleton)** — `resolveJourney`/`getJourneyTrace` that stitch traces via evidence-service; `txn_trace_map` in PG populated lazily.
9. **`taxonomy-service`** — rules table (`service, api, error_code, state` → taxonomy_id, template, action class, risk tier), Redis hot cache, gap logging to `tjsa-gap-v1`. Seed with UPI + IMPS SME workshop output.
10. **`kg-ingest` + `code-intelligence-indexer`** (batch, CI-triggered) — OpenAPI/Java contract ingestion into `api_registry`, service/dependency graph.
11. **`ui-knowledge-publisher`** (Node) + UI registry tables — RN/React screens/routes for payment journeys; first task procedures (send IMPS/NEFT/UPI, add beneficiary).
12. **`admin-api` (skeleton)** — read-only JourneyTrace viewer backend for ops (validates twin quality, W0.8).

### Infra this phase
`agent-platform` namespace, STRICT mTLS, AuthorizationPolicies, Workload Identity per service (only evidence-service gets BQ/Logging IAM), Istio VirtualService `/v1/tjsa/**` → stubs, NGINX TJSA rate bucket, CI/CD (lint, sqlc, migrate, testcontainers, image signing), Postgres/Redis/Neo4j/Temporal provisioning.

### Exit criteria (Blueprint VII.2)
≥95% trace reconstruction on sampled transactions for 2–3 payment journeys; taxonomy covers 100% of observed error codes on those journeys (4-week sample); PII guard and tool-gateway denial tests pass; audit hash chain verified; LLM hosting decision recorded (open question E-4).

---

## 4. Phase 1 — Explain-only MVP (12–16 weeks): EXPLAIN + GUIDE + INFORM at L0

The first customer-facing release: answers "why did my transfer fail / where is my money / how do I do X", gives deep links, raises tickets. **No actions.**

### New services

1. **`agent-gateway`** — `/v1/tjsa/converse` (SSE streaming), existing-session JWT validation, channel adapter (RN + React), conversation session store in Redis, per-device rate limits (complementing the NGINX bucket).
2. **`agent-orchestrator`** — the core: investigation state machine (§24.2), LLM planner loop with tool-call budget, response composer enforcing the II.5 template (money position first, mode-specific), intent classifier integration (II.2.3), clarify-on-low-confidence, escalation branch.
3. **`knowledge-service`** — pgvector RAG over approved KB (product facts, limits, cut-offs), source/version filters, citation of knowledge version.
4. **`ui-guidance-service`** — deep-link resolver against the UI registry, stale-route check, Task Guidance Engine (`getTaskGuidance`, `getNextStep`, `checkTaskPrerequisites`), guided-session store.
5. **`case-service`** — idempotent ticket creation with full investigation context attached, one adapter (bank's ticketing system).

### Extended
- `model-gateway` full: production LLM provider (per E-4 decision), structured-output JSON validation, fallback chain, cost metering.
- `transaction-intelligence` full: RCA classifier (rules-first, taxonomy lookup), financial state machine (debited/pending/reversed/refunded), `getFinancialState`.

### Registered tools this phase (tool-gateway)
`resolveJourney`, `getJourneyTrace`, `getFinancialState`, `classifyRootCause`, `getExplanation`, `getCustomerContext`, `getUIDeepLink`, `getTaskGuidance`, `checkTaskPrerequisites`, `getNextStep`, `getProductFact`, `createTicket`, `escalateToHuman`. All read-only + ticket create.

### Quality gates before ramp
Golden dataset v1 (≥200 cases) passing thresholds (VI.6); red-team suite v1 (prompt injection, PII exfiltration, out-of-scope) green; adversarial intents blocked; explanation correctness reviewed by SMEs.

### Rollout
Dark launch (shadow traffic) → employee cohort → 1% → 100% via the **existing device-id→bucket routing** (X.3). App ships chat sheet + "Explain this transaction" CTA + deep-link handler.

### Exit criteria (VII.3)
Explanation accuracy ≥ target on golden set; P95 latency within §26 budgets (first-turn BQ lazy lookups within async UX budget); zero PII incidents; kill switch drill executed.

---

## 5. Phase 2 — Safe corrective actions + expansion (12–20 weeks): L1

### New services

1. **`action-workflow` + `action-worker`** — Action Registry (versioned, risk-tiered), `listAvailableActions`/`executeAction`, Temporal workflows with idempotency ledger, confirmation + step-up (existing bank step-up service) for T2/T3, compensation logic. First actions: re-check status, resend confirmation, raise dispute (T1–T2, reversible).
2. **`evidence-ingest`** — the deferred T-2 tier (X.5.1): **filtered** Pub/Sub subscription (`event_class="payment_lifecycle" OR success="false"` ≈ 5–10% of firehose; ~600–1,200 events/s at the 12k/s peak), batched idempotent upserts into daily-partitioned PG (`txn_trace_map`, `transaction_event`), partition-drop retention job, HPA with low night minimums (C12). Powers the real-time lifecycle state machine and sub-second `resolveJourney` for recent payments.
3. **`journey-intelligence`** — Neo4j journey/dependency graph queries for multi-hop diagnosis.
4. **`incident-service`** — incident registry sync, impact correlation ("is this failure part of a known outage"), incident-aware response mode.

### Extended
- `agent-orchestrator`: async investigations (Temporal-triggered follow-ups, "I'll notify you when NEFT settles"), incident mode, action confirmation dialogs.
- Domain expansion: cards, beneficiaries, mandates — taxonomy entries, task procedures, KB content, deep links (content work on existing services).

### Exit criteria (VII.4)
Action success ≥ target with zero unsafe executions; idempotency proven under chaos tests (duplicate submits, worker crashes); T-2 freshness SLO met at daytime peak with bounded backlog; incident correlation validated in one real or staged incident drill.

---

## 6. Phase 3 — Deposits, loans, KYC/onboarding, disputes (12–14 weeks)

No new Go services — this phase proves the platform scales by **configuration and content**:

- Taxonomy entries + rules for new domains; KB documents + pgvector indexes; task procedures (FD creation, loan EMI queries, KYC re-submission); UI registry routes; new tool registrations for domain read APIs (via tool-gateway, no code change to the gateway itself); dispute workflows in `action-workflow` + `case-service` adapter extensions.
- Engineering work: load/perf hardening from Phase 1–2 telemetry, golden dataset v3 per domain, cost tuning (BQ scheduled aggregates, C6).

Exit: each new domain passes the same golden-set/red-team gates as payments did.

---

## 7. Phase 4 — Contact-centre copilot (8–10 weeks)

- `admin-api` extended into the **ops copilot backend**: agent-assist endpoints (same orchestrator, ops persona prompt from prompt-registry, wider evidence visibility per ops policy), warm-handoff context transfer.
- React ops console (non-Go) on the existing internal access path.
- `agent-gateway` gains an internal channel adapter (contact-centre identity via existing SSO).

Exit: handle-time reduction measured against control group; ops feedback loop feeding the golden dataset.

---

## 8. Phase 5 — Expanded autonomy L2 + learning loop (12+ weeks)

- `action-workflow`: L2 action classes (execute-then-notify for whitelisted T1 actions), expanded registry; `policy-service` bundles for L2 eligibility.
- **`learning-pipeline`** batch jobs (Go, night-trough scheduled per C12): taxonomy gap mining from `tjsa-gap-v1` + BQ, golden-set candidate mining, feedback aggregation → SME review queue in admin-api. Human-approved promotion only — no self-modifying knowledge (VI).
- Proactive support pilots (outbound "your NEFT settled") per the long-term roadmap (VII.7.1), gated by Risk Committee.

---

## 9. Cross-phase engineering rules (binding)

1. **Repo:** single Go monorepo; `internal/domain` has zero external imports; one binary per `cmd/`; sqlc + golang-migrate for all schemas; Buf for Connect/proto contracts.
2. **Testing pyramid per service** (V.17): unit → contract (tool JSON schemas vs handlers) → integration (testcontainers: PG, Redis, Pub/Sub emulator, Temporal test server) → golden-set evaluation → red team. CI blocks merge on any tier.
3. **Every tool call** goes orchestrator → tool-gateway → domain service. No service-to-LLM shortcuts; `model-gateway` has no data-store IAM (X.3).
4. **Every schema/prompt/policy/taxonomy change** is versioned, audited, and releasable independently of binaries.
5. **Diurnal ops** (C12): HPA night minimums, batch in the trough, bounded subscriber backlog at the 12k/s peak.
6. **Kill switches** tested every phase: tool-level (tool-gateway flag), agent-level (agent-gateway flag), route-level (Istio runbook).

## 10. Team shape (Blueprint VII.9)

| Role | Phase 0–1 | Phase 2+ |
|---|---|---|
| Go platform engineers | 4–6 | 6–8 |
| Data/BQ engineer | 1–2 | 1 |
| App engineers (RN/React, part-time) | 2 | 1–2 |
| SME/taxonomy content (payments ops) | 2 | 2–3 per domain |
| AI/prompt engineer | 1 | 2 |
| QA/eval engineer | 1 | 2 |
| SRE/platform | 1–2 | 2 |

---

*Plan version 1.0 — re-baseline durations after Phase 0 discovery (Blueprint VII.1). All open questions E-1…E-9 and I.11 #1–#3 need owners before Phase 0 kick-off.*
