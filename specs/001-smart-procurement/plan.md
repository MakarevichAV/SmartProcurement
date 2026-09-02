# Implementation Plan: Smart Procurement

**Branch**: `001-smart-procurement` | **Date**: 2026-09-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-smart-procurement/spec.md`

## Summary

Smart Procurement is an AI decision layer over enterprise data that observes procurement
signals, diagnoses shortage/quality/price risks with explanations, produces structured buying
recommendations, and — gated by a LORM responsibility layer — either shows the recommendation
(L3), asks a human to approve a prepared action (L4), or executes autonomously inside an
approved machine-readable policy (L5). Every L4/L5 action is verified against configurable
tolerances and audited; capabilities demote automatically on incident/verification-failure/
observability-loss/policy-expiry.

**Technical approach (v1, diploma-realistic):** a Python/FastAPI **modular monolith** backend
plus a **worker process** sharing one codebase; **PostgreSQL** for all state including a
**durable Postgres-backed job queue** (no external broker); a **React/TypeScript** SPA in a
separate repo talking to the backend over a REST/OpenAPI contract. The official
[LORM reference implementation](https://github.com/Argyronix/lorm) supplies the **canonical
policy schema and validator** (vendored, Apache-2.0); its Claude-Code hook enforcement is
re-implemented as an in-process `LormEnforcementService` that preserves the same
`allow / ask / deny` decision semantics and invariants I-1…I-8. The LLM sits behind an
`LLMProvider` seam, always returns schema-validated structured output, and can never trigger
execution directly.

## Technical Context

**Language/Version**: Backend Python 3.12; Frontend TypeScript 5.x on Node 20.

**Primary Dependencies**:
- Backend: FastAPI, Pydantic v2, SQLAlchemy 2.0 (async), Alembic, `anthropic` SDK (LLM),
  `jsonschema>=4` + `PyYAML` (vendored LORM validator), `cryptography` (secret encryption),
  `argon2-cffi` (password hashing), `python-jose`/`pyjwt` (JWT), `httpx` (outbound REST),
  `pytest`/`pytest-asyncio`.
- Frontend: React 18, Vite 5, React Router, Redux Toolkit + RTK Query, Tailwind CSS,
  Vitest + React Testing Library, Playwright (E2E).

**Storage**: PostgreSQL 16, single database. `pgvector` extension enabled but used **only**
for semantic retrieval over unstructured supplier/quality/document text (opt-in per source);
not a substitute for the relational procurement model. No separate vector DB in v1.

**Testing**: pytest (backend unit/integration/contract), httpx AsyncClient (API), dockerised
PostgreSQL for integration, Vitest + RTL (frontend), Playwright (end-to-end demo), k6 or
Locust (load test at the SC-016 target). LLM-dependent tests use a `DeterministicMockProvider`
so the default suite needs no external LLM.

**Target Platform**: Linux server (backend API + worker as two processes), modern browser SPA.

**Project Type**: Web application — separate `backend/` and `frontend/` git repos inside one
local `SmartProcurement/` workspace; specs/docs in the root repo.

**Performance Goals** (from SC-016, single enterprise): sustain ~100 000 observation
signals/day with no growing backlog in the observation loop; detected risks visible on the
Dashboard within the source's configured observation interval (default 15 min); non-AI API
endpoints p95 < 500 ms; recommendation generation p95 < 30 s (LLM-bound, async job).

**Constraints**:
- One active enterprise per deployment (multi-enterprise-ready data model).
- No external message broker (Kafka/RabbitMQ/Redis-broker) in v1 — durability via Postgres.
- Deterministic preprocessing/aggregation/filtering **before** any LLM call; the LLM is never
  invoked per raw signal.
- LLM output is always validated against a typed schema; invalid/timeout/unavailable →
  fail-safe path (retry/backoff → degraded "human-required" → block dependent L5 → audit).
- Backend is the only security boundary; frontend carries no trusted authz/LORM logic.
- Audit store is append-only from the application's perspective and independent of LLM context.
- No agent framework (LangChain/LangGraph) unless a concrete need is proven in design.

**Scale/Scope**: ≤ 10 000 active SKUs, ≤ 10 connected data sources, ≤ 100 000 signals/day per
enterprise deployment; 9 UI areas; 8 procurement capabilities; 3 roles.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Constitution v1.0.0 — evaluation of each principle against this plan:

| Principle | Plan compliance |
|-----------|-----------------|
| **I. Назначение и границы продукта** | Decision layer only; `execution/` dispatches through swappable adapters; no ERP/WMS/MES ledger, warehouse, accounting or payment-execution features (FR-074). `proc.payment.release` stays "prepare, never release". ✅ |
| **II. LORM как модель ответственности** | LORM level is per **Capability**, not system-wide (`lorm/` module, `capability` table). L0–L5 semantics implemented as observation → diagnosis(+uncertainty) → recommendation(+expected/risks/alternatives/do-nothing) → ask → policy-bounded autonomy. ✅ |
| **III. Переиспользование LORM Reference Implementation** | Vendor `schema/lorm-policy.schema.json` + `validate_policy.py` unchanged (Apache-2.0) as the canonical policy schema/validator. Adaptation layer (`LormEnforcementService`) justified because Claude-Code `hooks/` cannot run inside a FastAPI runtime; it preserves official `allow/ask/deny` semantics and I-1…I-8. See research.md §6 and [Complexity Tracking](#complexity-tracking) (no violation — permitted adapter per §3). ✅ |
| **IV. Capability-based architecture** | 8 capabilities modelled explicitly with independent `level`; `proc.supplier.add` and `proc.payment.release` hard-capped below L5 in code and schema-seed (FR-049). ✅ |
| **V. Человек сохраняет ответственность и контроль** | Promotion is human-only, one level at a time, API returns `PromotionRequest` (never auto-applied). Automatic **demotion** without approval on LORM §6.3 triggers (`lorm/demotion.py`). AI self-promotion impossible: no code path writes `capability.level` upward. ✅ (I-8) |
| **VI. L5 допускается только через формальную Policy** | `policies/` stores canonical machine-readable policy matching the LORM schema (capability, bounds, version, author, approved_by ≠ author, approved_at, expires, revocation, verification + tolerances). AI can draft (`PolicyDraft`), never approve/activate. State machine: draft → pending_approval → active → (expired|revoked). ✅ |
| **VII. AI принимает решение, LORM определяет полномочия** | `analysis/`+`decision/` produce the buying decision; `lorm/` only authorizes. Every state-changing path calls `LormEnforcementService.evaluate()` before `execution/`. No bypass: `execution.dispatch()` requires an `EnforcementDecision` token. ✅ |
| **VIII. Отделение принятия решения от исполнения** | `Recommendation` / `ProcurementAction` are ERP-independent structured records; `ExecutionAdapter` is a Protocol with ≥2 v1 implementations (simulated, generic REST); decision modules import no adapter code. ✅ (SC-014) |
| **IX. Универсальность и адаптация к предприятию** | `SourceConnector` Protocol; `FieldMapping` persisted as explicit, human-confirmed config; AI mapping suggestions require confirmation (FR-004). No ERP-vendor coupling in domain model. ✅ |
| **X. Аудитируемость** | `audit/` is a first-class append-only subsystem; `AuditRecord` carries the LORM §10.3 minimum set + spec FR-060 fields; recovery/verification/demotion events recorded; store is Postgres, not LLM context. ✅ |
| **XI. Безопасность** | Backend-enforced JWT auth + RBAC (FastAPI dependencies); external-source credentials encrypted at rest (`cryptography.Fernet`), key from env/secret store, never in plaintext columns, never in prompts (FR-068). LORM enforcement and RBAC are separate, both required. Frontend untrusted. ✅ |
| **XII. Объяснимость решений** | `Explanation` entity (what/why/data-used/factors/confidence) attached to every `RiskFinding` and `Recommendation`; low-confidence → escalate, never autonomous (I-3, FR-025). ✅ |
| **XIII. Архитектурное разделение** | Modular monolith with explicit module interfaces mapping 1:1 to the constitution's layers (see research.md §2). `api/`=Frontend boundary, `analysis/`+`ai/`=AI Decision Layer, `lorm/`=Enforcement, `integration/`=Data/Integration, `execution/`=Execution, SQLAlchemy models + `audit/`=Persistence. Modules communicate through service interfaces, not by reaching into each other's tables. ✅ |
| **Архитектурное разделение слоёв (section)** | Same as XIII; `ai/` cannot call `execution/`; `execution.dispatch()` guarded by an enforcement token. ✅ |
| **Зрелость и эволюция продукта (section)** | Seams: `LLMProvider`, `ExecutionAdapter`, `SourceConnector`, `SecretStore`, `AuthProvider`, Postgres-backed `JobQueue`. New capability/source/adapter = new row + new class, no core rewrite (SC-015). Multi-enterprise-ready schema (nullable-free `enterprise_id` FK everywhere). ✅ |

**Result: PASS.** No principle is violated; no entry in Complexity Tracking. The single
adaptation (re-implementing enforcement instead of running Claude-Code hooks) is explicitly
sanctioned by Constitution §3 and documented in research.md §6.

*Re-check after Phase 1 design: **PASS** — data-model.md and contracts/ keep decision logic
free of adapter/ERP types, keep `capability.level` write-once-upward-by-human, and carry the
LORM §10.3 audit fields. No new violations introduced.*

## Project Structure

### Documentation (this feature)

```text
specs/001-smart-procurement/
├── plan.md              # This file (/speckit-plan command output)
├── spec.md              # Feature specification + clarifications
├── research.md          # Phase 0 output — decisions & rationale
├── data-model.md        # Phase 1 output — entities, relationships, state machines
├── quickstart.md        # Phase 1 output — end-to-end validation guide
├── contracts/           # Phase 1 output — interface contracts
│   ├── rest-api.md          # Frontend↔Backend REST/OpenAPI contract
│   ├── ai-structured-output.md  # Typed LLM output schemas
│   ├── execution-adapter.md     # ExecutionAdapter Protocol + models
│   ├── source-connector.md      # SourceConnector Protocol + models
│   └── lorm-enforcement.md      # EnforcementRequest/Decision + canonical Policy
└── tasks.md             # Phase 2 output (/speckit-tasks — NOT created here)
```

### Source Code (local workspace `SmartProcurement/`)

The root directory is its own git repo. `backend/` and `frontend/` are **separate git repos**
(`SmartProcurement-Backend`, `SmartProcurement-Frontend`) and are listed in the root
`.gitignore`; no submodules. Frontend↔Backend interact only through the REST contract.

```text
SmartProcurement/                     # root repo: docs + Spec Kit artifacts only
├── README.md                         # product + architecture overview, repo links, LORM role, run instructions
├── .gitignore                        # ignores: frontend/  backend/  .env  .claude/ locals
├── .specify/  specs/  .claude/
├── docs/
│   ├── product-overview.md
│   ├── architecture.md
│   ├── api-integration.md
│   ├── lorm-integration.md           # reuse vs adapt vs build split
│   ├── development-setup.md
│   ├── repository-map.md
│   └── adr/                          # ADRs for non-obvious decisions
│
├── backend/                          # repo: SmartProcurement-Backend
│   ├── src/app/
│   │   ├── api/                      # FastAPI routers + request/response schemas (OpenAPI source)
│   │   ├── core/                     # config (pydantic-settings), security, errors, db session, secret store
│   │   ├── identity/                 # users, roles, permissions, JWT auth, RBAC deps
│   │   ├── enterprise/               # enterprise configuration
│   │   ├── integration/              # SourceConnector impls, schema introspection, FieldMapping, sync
│   │   ├── domain/                   # canonical procurement entities + repositories (L0 map)
│   │   ├── observation/             # signal ingestion, normalization, rolling aggregates, schedule triggers (L1)
│   │   ├── analysis/                 # deterministic risk rules + AI explanation orchestration (L2)
│   │   ├── decision/                 # recommendation assembly, ProcurementAction builder (L3)
│   │   ├── lorm/                     # LormEnforcementService, capability/level store, demotion engine
│   │   ├── policies/                 # L5 policy CRUD, approval workflow, vendored schema validation
│   │   ├── execution/               # ExecutionAdapter impls, dispatch, reconciliation (L4/L5)
│   │   ├── verification/            # verification runners, tolerance evaluation
│   │   ├── audit/                    # append-only AuditRecord writer + query
│   │   ├── jobs/                     # Postgres-backed durable job queue + worker loop + scheduler
│   │   ├── ai/                       # LLMProvider protocol, Anthropic impl, DeterministicMockProvider, output schemas
│   │   └── vendor/lorm/              # vendored: lorm-policy.schema.json, validate_policy.py, SPEC.md (Apache-2.0, unmodified)
│   ├── alembic/                      # migrations
│   ├── tests/                        # unit/ integration/ contract/ e2e-support/
│   └── pyproject.toml
│
└── frontend/                         # repo: SmartProcurement-Frontend
    ├── src/
    │   ├── app/                      # Redux store, router
    │   ├── api/                      # RTK Query slices, types generated from backend OpenAPI
    │   ├── features/                 # dashboard/ risks/ recommendations/ approvals/ policies/
    │   │                             #   capabilities/ datasources/ executions/ audit/ users/
    │   ├── components/               # shared UI (Tailwind)
    │   └── lib/
    ├── tests/                        # Vitest + RTL
    ├── e2e/                          # Playwright — primary procurement scenario
    └── package.json
```

**Structure Decision**: Web application, three repos as mandated by the plan input.
Backend is a **modular monolith** (`src/app/<module>/`) where each module owns its tables and
exposes a service interface; the constitution's layers are enforced by module boundaries and
an enforcement-token guard on `execution.dispatch()`, not by separate deployables. A second
process (`python -m app.worker`) runs the job queue/scheduler against the same code and DB.
This is the smallest structure that satisfies layer separation, background operation,
restart recovery, and future component extraction without microservices.

## Complexity Tracking

> No Constitution Check violations. Table intentionally empty.

The one adaptation — an in-process `LormEnforcementService` instead of the reference
implementation's Claude-Code `hooks/` — is **not** a violation: Constitution §3 explicitly
permits an adaptation layer "если отдельные компоненты reference implementation специфичны для
Claude Code … при сохранении семантики и требований официальной спецификации LORM." The
canonical schema and validator are reused unchanged. Rationale and the reuse/adapt/build split
are in `research.md` §6 and will be mirrored in `docs/lorm-integration.md`.
