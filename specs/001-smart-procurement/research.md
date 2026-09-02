# Phase 0 Research — Smart Procurement

All decisions below are scoped to **v1 (diploma-realistic)** while keeping the seams needed
to grow into a real enterprise product. Each decision lists rationale and rejected
alternatives. Items marked *(input)* were fixed by the plan input; only their rationale is
recorded.

---

## 1. Repository & workspace layout *(input)*

**Decision**: Three git repos — `SmartProcurement` (root: README, `.specify/`, `specs/`,
`docs/`, `.claude/`), `SmartProcurement-Frontend`, `SmartProcurement-Backend`. The root repo
`.gitignore`s `frontend/` and `backend/`; no submodules. Spec Kit runs from the root and
describes the whole system.

**Rationale**: Independent deploy/versioning of FE and BE; a clean project-level home for
governance artifacts; Claude Code operates on the whole workspace locally while each app keeps
its own history. Submodules add checkout/CI friction with no v1 benefit.

**Rejected**: Monorepo (couples FE/BE history, not wanted); git submodules (pinning overhead).

---

## 2. Backend architecture — modular monolith + worker

**Decision**: One FastAPI application, `src/app/<module>/`, each module owning its tables and
exposing a service class. A second OS process `python -m app.worker` runs the scheduler + job
queue against the same code/DB. Modules map to constitution layers:

| Constitution layer | Module(s) |
|---|---|
| Frontend boundary | `api/` (routers + DTOs only) |
| Backend/API orchestration | `api/`, per-module services |
| AI Decision Layer | `analysis/`, `decision/`, `ai/` |
| LORM Responsibility/Enforcement | `lorm/`, `policies/` |
| Data/Integration | `integration/`, `domain/`, `observation/` |
| Execution | `execution/` |
| Persistence | SQLAlchemy models per module, `audit/`, `jobs/` |

**Boundary rules enforced in code**: `ai/` and `analysis/` import nothing from `execution/`;
`execution.dispatch(action, decision_token)` refuses to run without a valid
`EnforcementDecision` produced by `lorm/`; cross-module reads go through service interfaces,
never foreign `SELECT`s.

**Rationale**: Meets Constitution XIII with one deployable + one worker — the least machinery
that still lets a module be extracted later. Diploma-scale (≤10 k SKUs) does not justify
service decomposition.

**Rejected**: Microservices (Constitution §3/§14 "no microservices without proven need");
single-process app with in-thread background tasks (cannot survive restart → violates FR-032a).

---

## 3. Background processing — Postgres-backed durable queue + scheduler

**Decision**: A single `job` table (`id`, `kind`, `payload jsonb`, `run_at`, `status`,
`attempts`, `max_attempts`, `locked_by`, `locked_at`, `last_error`, timestamps). The worker
loop:

```
SELECT ... FROM job
WHERE status='pending' AND run_at <= now()
ORDER BY run_at
FOR UPDATE SKIP LOCKED
LIMIT N;
```

Handlers are idempotent; failures reschedule with exponential backoff
(`run_at = now() + base * 2**attempts`, capped). Recurring observation is self-perpetuating:
each `observe_source` job, on completion, enqueues the next at `now() + source.interval`.
Job enqueue happens **in the same DB transaction** as the domain write that triggers it.

**Rationale**: Durability, retry/backoff, scheduling, and crash recovery all fall out of
Postgres with ~150 LOC and zero extra infrastructure — directly reusing the persistence store
FR-032a already mandates. `SKIP LOCKED` is battle-tested for this. Target load (~1–2 jobs/s
average, low tens/s peak) is trivially within a single worker's reach.

**Rejected**:
- **Celery + Redis/RabbitMQ**: extra broker + result backend to run, monitor, and secure;
  at-least-once anyway (still need idempotency); heavy for a diploma. Constitution §12 says
  don't add heavy distributed infra without need.
- **APScheduler with in-memory jobstore**: not durable across restart.
- **APScheduler + SQLAlchemyJobStore**: durable scheduling but no work-queue semantics, retry,
  or `SKIP LOCKED` fan-out — we'd still build the queue.
- **arq / Dramatiq / RQ**: all pull in Redis.

**Scaling note**: horizontal workers already work (SKIP LOCKED); a broker can replace the
table later behind the same `JobQueue` interface if throughput ever demands it.

---

## 4. Authentication & authorization

**Decision**: Local username/password with **Argon2id** hashing; **JWT** access tokens
(~15 min) + rotating refresh tokens (persisted, revocable). RBAC = `role → permission` map;
FastAPI dependencies (`require(permission)`) guard every route; LORM enforcement is a
**separate** check on top for state-changing procurement actions. Auth logic sits behind an
`AuthProvider` interface.

**Rationale**: Stateless, standard, no external IdP to stand up; works cleanly for an SPA on a
different origin (bearer token, no CSRF surface). Satisfies FR-064–FR-067 and Constitution XI.
`AuthProvider` seam lets OIDC/SSO drop in later.

**Rejected**: Full OAuth2/OIDC + external IdP (infra/config overhead, no v1 value);
server-side session cookies (cross-origin CSRF handling, shared session store).

**Separation-of-duties (FR-038/FR-065)**: enforced in `policies/` service —
`policy.author_id != policy.approved_by_id` checked at approval time regardless of roles;
covered by an explicit test.

---

## 5. Secret & configuration management

**Decision**: `pydantic-settings` reads config from **environment variables** (dev: a
git-ignored `.env`; deploy: injected env). External data-source and execution-endpoint
credentials are stored in Postgres **encrypted with `cryptography.Fernet`**; the Fernet key
comes from the environment/secret store, never the DB. A `SecretStore` interface wraps
get/put so a managed secret manager can replace it later. Credentials are never logged, never
placed in prompts, never returned by the API (write-only fields).

**Rationale**: Satisfies FR-068 and Constitution §15 with no extra infrastructure; the seam
keeps Vault/cloud-KMS a drop-in.

**Rejected**: HashiCorp Vault / cloud secret manager for v1 (operational overhead for a
diploma); plaintext credential columns (violates §15).

---

## 6. LORM integration — reuse vs adapt vs build

Source studied: `Argyronix/lorm` — `SPEC.md` (v2.0.2), `schema/lorm-policy.schema.json`
(v1.5), `skills/lorm/scripts/validate_policy.py`, `hooks/` (PreToolUse gate + PostToolUse
audit), `examples/procurement.md`, `docs/trust-lifecycle.md`, `docs/hard-enforcement.md`.

### 6a. Reused directly (vendored unmodified into `backend/src/app/vendor/lorm/`, Apache-2.0)

| Artifact | Use in Smart Procurement |
|---|---|
| `schema/lorm-policy.schema.json` | The **canonical** shape of a stored L5 policy. Our DB row serializes to a document that validates against this schema. |
| `skills/lorm/scripts/validate_policy.py` | Run on every policy create/version/approve; reject on schema failure (FR-036, FR-044). |
| `SPEC.md` §6.3 demotion triggers, §6.2 promotion rules, §10.1 verification, §10.3 audit fields, invariants I-1…I-8 | Directly implemented as service logic and test oracles. |
| `schema/examples/*.lorm-policy.yaml`, `examples/procurement.md` | Seed data for the 8 capabilities and the demo L5 replenishment policy. |

### 6b. Claude-Code-specific — NOT usable in a FastAPI runtime

`hooks/` (PreToolUse/PostToolUse) are Claude-Code tool-call interceptors keyed to `Bash`,
`Write`, `Edit`, and `mcp__*` tool names with fnmatch `match:` patterns. Our state changes are
domain operations (create PO, add supplier), not Claude tool calls. The `skills/lorm/` agent
skill is soft guidance for an LLM agent, not a backend gate.

### 6c. Built in Smart Procurement — `LormEnforcementService` (adaptation layer)

An in-process service preserving official semantics:

- **Input** `EnforcementRequest{ capability_id, proposed_action, evidence_ref (L2 diagnosis),
  uncertainty, enterprise_id }`.
- **Output** `EnforcementDecision{ decision: allow|ask|deny, effective_level, reason,
  policy_id?, policy_version? }` — the same three outcomes the reference PreToolUse hook
  returns (`allow` / `ask` / `deny`).
- **Rules**: I-1 (no action without an L2 diagnosis grounded in L1) → `deny` if `evidence_ref`
  missing/stale; capability `level` caps the outcome (`L≤3` → `deny`/recommend-only, `L4` →
  `ask`, `L5` → evaluate policy); L5 → load **active** policy for the capability, check
  `expires`, `revoked`, bounds (SKU/category, suppliers, per-order cap, period aggregate from
  audit log), and executable `conditions[].check`; any bound breached or condition failing →
  degrade to `ask` (L4) — never silently execute (FR-042); `uncertainty` above the policy/
  proposal threshold → degrade + flag (I-3, FR-025).
- **No bypass**: `execution.dispatch()` signature requires the `EnforcementDecision`; there is
  no other entry point.

### 6d. Demotion — automatic at runtime, human-curated in the policy file

`SPEC.md` §6.3: on incident / rollback / unexplained verification failure / telemetry
(observability) loss / uncertainty over threshold / policy expiry|revocation, "the affected
capability MUST immediately drop one level … without waiting for human review." Invariant I-8
("propose, never enact") applies to **upward** changes and to editing the policy document's
`demotions:` array — not to the mandatory automatic downward step.

**Implementation**: `lorm/demotion.py` writes a `DemotionEvent` and decrements
`capability.level` by exactly one, transactionally, triggered by `verification/`, `analysis/`
(uncertainty, observability loss), `execution/` (incident/rollback), and a scheduled
`policy_expiry_scan` job. Promotion is only ever a `PromotionRequest` requiring a human
(and, for L4→L5, a human other than the policy author).

**Rationale**: This is the only way to run LORM enforcement inside a Python web backend;
Constitution §3 sanctions it; semantics and invariants are preserved and test-pinned.

**Rejected**: Shelling out to the Claude-Code hooks (wrong runtime, wrong trigger surface);
writing our own policy schema (Constitution §3 forbids an incompatible reimplementation).

---

## 7. AI Decision Layer

**Decision**: `LLMProvider` protocol —
`generate_structured(output_schema: type[BaseModel], system, prompt, context) -> BaseModel`.
v1 implementations: `AnthropicProvider` (uses the `anthropic` SDK; default to the latest
Claude model) and `DeterministicMockProvider` (fixture-driven, used by the default test
suite). All AI results consumed downstream are Pydantic models; a validation failure is
treated exactly like unavailability (§8). No LangChain/LangGraph — orchestration is plain
Python (`analysis/`, `decision/`), a few sequential provider calls; revisit only if a proven
need appears.

AI tasks: source-schema interpretation & `MappingSuggestion`; risk `Explanation`;
situation analysis and supplier/option comparison; `ProcurementRecommendation`; `PolicyDraft`
assistance. AI **never** calls `execution/` and never writes `capability.level`.

**Rationale**: Provider abstraction satisfies Constitution "replace LLM provider"; typed
output satisfies FR-071 and "validate via typed schemas"; mock keeps the suite free of
external-LLM cost/flakiness (FR test requirement).

**Rejected**: Free-text LLM responses feeding execution (forbidden, §5/§7); agent framework
by default (§5 "add only if concretely necessary").

### 7a. Cost/scale guard (SC-016, §16)

Pipeline: raw signal → normalize → per-SKU rolling aggregates (consumption rate, days-of-
cover, price series, supplier-delay series) → **deterministic** risk-candidate rules → only a
candidate crossing a threshold spawns an AI explanation/recommendation job, deduplicated per
open `RiskFinding`. The LLM is never called per raw signal. Forecasting in v1 = moving-average
consumption projection vs lead time (no ML).

---

## 8. LLM failure handling *(input; from Clarifications)*

**Decision**: retry with exponential backoff → on persistent failure/invalid output, mark the
`RiskFinding`/`Recommendation` `ai_status = unavailable` ("human-required"), surface on
Dashboard, **block dependent L5** execution, write an audit entry. No autonomous action on
stale/absent AI output. Treated as observability loss for demotion purposes (FR-016a, §6.3).

**Rationale**: Matches Clarifications answer and LORM telemetry-loss semantics.

**Rejected**: rule-based fallback recommendation (second decision path to build/verify; not
needed for v1); infinite silent queue (invisible delays on time-sensitive shortages).

---

## 9. Reliable execution & restart recovery *(input; from Clarifications / FR-032a)*

**Decision**: Every `ProcurementAction` gets a UUID + a stable `idempotency_key`. State is
persisted **before** `execution.dispatch()`. Adapters accept the idempotency key and expose
`get_status(idempotency_key)`. On worker startup a `reconcile_executions` job:
`sent_unknown` actions → `adapter.get_status()` (never blind resend); `authorized`/`prepared`
but undispatched durable actions → re-enqueued; every recovery step → `AuditRecord`.

**Rationale**: No double purchase orders, no dropped authorized actions; recovery is auditable.

**Rejected**: fire-and-forget resend (duplicate POs); "human review everything on restart"
(pauses autopilot, manual load after any restart).

---

## 10. Verification & configurable tolerances *(input; from Clarifications / FR-055a)*

**Decision**: `verification/` compares observed vs expected on price (% deviation), delivered
quantity (% short), lead time (days late), and supplier identity (any mismatch = material).
Tolerances live in `capability.verification_tolerances` (JSONB) with conservative defaults,
optionally overridden per `policy`. Result: `verified | failed | unverifiable | pending`, with
per-dimension measured deviation and a `material` flag. `failed` (material, unexplained) →
`lorm/demotion.py`. Repeated `unverifiable` → surfaced to the approver, blocks L5 eligibility
(SPEC §10.1). A `verification_window` per action bounds the wait; window elapsed with no data
→ `unverifiable`.

**Rejected**: hard-coded global constants (§13 forbids); zero-tolerance (autopilot demotes on
trivial real-world variance).

---

## 11. Data integration

**Decision**: `SourceConnector` protocol — `test_connection()`, `describe_schema()`,
`fetch(since)`. v1 connectors: `RestSourceConnector` (JSON over HTTP, configurable
endpoint/auth), `FileSourceConnector` (CSV/JSON upload), `SqlSourceConnector` (read-only DSN,
SELECT-only). `describe_schema()` output feeds an AI `MappingSuggestion`; the admin confirms
before a `FieldMapping` becomes active (FR-004/FR-005). Source health (`available` /
`unavailable` / `stale`) tracked from each `fetch`/`test_connection` and shown per L0 entity.

**Rationale**: Covers the spec's source types (ERP/MES/WMS/DB/API/files) without ERP-vendor
coupling; explicit persisted mappings satisfy Constitution IX.

**Rejected**: a fixed ERP schema assumption (violates IX); auto-applying AI mappings
(violates FR-004).

---

## 12. Execution layer *(input)*

**Decision**: `ExecutionAdapter` protocol —
`dispatch(command, idempotency_key) -> DispatchResult`, `get_status(idempotency_key) ->
ExecutionStatus`. v1: `SimulatedExecutionAdapter` (deterministic, the demo default,
configurable latency/outcomes for tests) and `GenericRestExecutionAdapter` (real HTTP POST to
a configurable external/stand-in endpoint, maps HTTP errors into the unified error model).
Adapter chosen per enterprise config. ERP-specific and RPA adapters are future work behind the
same protocol.

**Rationale**: FR-031a exactly; proves swappability (SC-014) without a commercial ERP.

---

## 13. Persistence & pgvector

**Decision**: PostgreSQL 16, one DB, SQLAlchemy 2.0 async + Alembic. `pgvector` extension
enabled; an `embedding vector` column is added **only** to unstructured-text tables
(supplier notes, quality/defect reports, attached documents) for semantic retrieval that
feeds AI context. Structured procurement data stays fully relational. Every table carries
`enterprise_id` (FK, non-null) for multi-enterprise readiness even though v1 runs one.

**Rationale**: Matches plan input; avoids a second datastore; keeps the relational model
authoritative (§4).

**Rejected**: dedicated vector DB (Constitution/plan say no without proven need); embeddings
as a substitute for relational modelling (§4 forbids).

---

## 14. API & contracts

**Decision**: REST/JSON, OpenAPI auto-generated by FastAPI. Pydantic request/response DTOs in
`api/` are defined **before** the UI that depends on them; the frontend generates TS types
from the published OpenAPI document. One unified error model
(`{ error: { code, message, details?, correlation_id } }`); external-integration failures are
mapped into it. Contracts for AI output, execution adapters, source connectors, and LORM
enforcement are versioned in `specs/001-smart-procurement/contracts/`.

**Rejected**: GraphQL (no v1 need, extra tooling); hand-written OpenAPI (FastAPI generates it).

---

## 15. Testing strategy

**Decision**: pytest (unit/integration/contract) with a dockerised Postgres; httpx AsyncClient
for API tests; `DeterministicMockProvider` for all LLM-dependent tests in the default suite; a
separate opt-in suite exercises the real provider. Dedicated suites: policy schema validation,
L4 approval flow, L5 autonomous flow, demotion triggers, adapter behaviour,
idempotency/restart recovery, AI structured-output validation & failure. Frontend: Vitest +
RTL for components/flows. E2E: Playwright drives the primary procurement scenario at L3/L4/L5.
Load: k6/Locust against the SC-016 target with the simulated adapter and mock provider.

**Rationale**: Directly covers spec §Testing; keeps the core suite deterministic and free of
external-LLM cost/availability.

---

## 16. Open items intentionally deferred to `/speckit-tasks` or implementation

- Exact default tolerance numbers (price %, qty %, days) — pick conservative values in
  implementation, expose as config; not architecture.
- Precise Dashboard layout / navigation (FR-073 allows UX refinement).
- Pagination/sort defaults for list endpoints — standard cursor pagination, decide per
  endpoint in contracts detailing.
- Localization/i18n of the UI — English + Russian resource files planned; not a blocker.
