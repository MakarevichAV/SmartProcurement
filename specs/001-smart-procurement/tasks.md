---

description: "Task list for Smart Procurement v1 implementation"
---

# Tasks: Smart Procurement

**Input**: Design documents from `/specs/001-smart-procurement/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md,
`.specify/memory/constitution.md`

**Tests**: INCLUDED — the plan input §18 explicitly requires unit / domain-LORM / API /
policy-validation / L4 / L5 / demotion / adapter / idempotency-recovery / AI-output /
frontend / E2E / load tests. Test tasks precede the implementation they cover.

**Organization**: grouped by user story (US1–US9 from spec.md, priority order). Each story
phase is an independently testable increment.

**Repos** (plan §1): `backend/` = `SmartProcurement-Backend`, `frontend/` =
`SmartProcurement-Frontend`, both standalone git repos inside this workspace, git-ignored by
the root. Paths below are relative to the workspace root.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: US1–US9; omitted for Setup / Foundational / Polish

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Initialize both repositories and local tooling.

- [X] T001 Create `backend/` repo: `git init`, `pyproject.toml` (Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0[async], Alembic, anthropic, cryptography, argon2-cffi, pyjwt, httpx, jsonschema, PyYAML, pytest, pytest-asyncio), package skeleton `backend/src/app/` with empty modules `api core identity enterprise integration domain observation analysis decision lorm policies execution verification audit jobs ai vendor` per plan.md Project Structure
- [X] T002 [P] Configure backend tooling in `backend/pyproject.toml`: ruff + black + mypy; add `backend/.editorconfig`; add `backend/Makefile` targets `lint test run worker migrate seed`
- [X] T003 [P] Create `frontend/` repo: `git init`, Vite + React 18 + TypeScript, Tailwind CSS, ESLint + Prettier; base folders `frontend/src/{app,api,features,components,lib}` per plan.md
- [X] T004 [P] Add `docker-compose.yml` at workspace root for local PostgreSQL 16 (plain — `pgvector` is not required in v1); document ports in `docs/development-setup.md`
- [X] T005 Backend config module `backend/src/app/core/config.py` using `pydantic-settings` (DATABASE_URL, JWT_SECRET, FERNET_KEY, LLM_PROVIDER, ANTHROPIC_API_KEY, token TTLs); add `backend/.env.example`
- [X] T006 Backend async DB layer `backend/src/app/core/db.py` (async engine, session factory, `Base`); initialize Alembic in `backend/alembic/` with async env
- [X] T007 [P] Backend base model mixins in `backend/src/app/core/models.py`: `UUIDPrimaryKey`, `Timestamps`, `EnterpriseScoped` (non-null `enterprise_id` FK), `AppendOnly` marker; helper for CHECK-constrained enum columns
- [X] T008 [P] Backend error handling in `backend/src/app/core/errors.py` + `backend/src/app/api/middleware.py`: unified error model `{error:{code,message,details,correlation_id}}`, exception handlers, `X-Correlation-Id` middleware
- [X] T009 [P] Backend test harness `backend/tests/conftest.py`: dockerised-Postgres fixture, async test client (httpx `AsyncClient`), transaction-rollback fixture, `DeterministicMockProvider` fixture registry under `backend/tests/fixtures/`
- [X] T010 [P] Frontend app shell `frontend/src/app/store.ts` (Redux Toolkit), `frontend/src/app/router.tsx` (React Router), `frontend/src/api/baseApi.ts` (RTK Query, bearer-token base query, unified-error normalization); add `frontend/scripts/gen-api-types.ts` to generate types from backend `/openapi.json`

**Checkpoint**: both apps build and run an empty health route.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Cross-cutting infrastructure every user story depends on — identity/auth,
enterprise + capability + **minimal capability-promotion** + audit + append-only guard +
job-queue + LLM seams. Per plan §20 the domain/LORM/contract boundaries are established here
before AI workflows and UI.

**⚠️ CRITICAL**: no user-story work begins until this phase is complete.

### Identity, auth, RBAC

- [X] T011 [P] Identity models in `backend/src/app/identity/models.py`: `user`, `role`, `permission`, `role_permission`, `user_role`, `refresh_token` (data-model.md §1)
- [X] T012 Alembic migration for identity + `enterprise` tables in `backend/alembic/versions/`
- [X] T013 Password + token services `backend/src/app/identity/security.py`: Argon2id hash/verify, JWT issue/verify (access + refresh), refresh-token rotation & revocation
- [X] T014 `AuthProvider` protocol + local implementation in `backend/src/app/identity/auth_provider.py`; FastAPI deps `get_current_user` and `require(permission)` in `backend/src/app/identity/deps.py`
- [X] T015 Auth router `backend/src/app/api/routers/auth.py`: `POST /api/v1/auth/login|refresh|logout`, `GET /api/v1/me` (user + effective permissions); `login`/`refresh` return the access token in the body and set the refresh token as an **httpOnly, Secure, SameSite** cookie; `logout` clears the cookie and revokes the token; wire into `backend/src/app/main.py`
- [X] T016 [P] Auth contract tests `backend/tests/contract/test_auth.py` (login/refresh/logout/me, 401 paths) per contracts/rest-api.md

### Enterprise, secrets

- [X] T017 [P] `enterprise` model + service in `backend/src/app/enterprise/` (`base_currency`, `active_execution_adapter`, `settings`); router `GET/PATCH /api/v1/enterprise`
- [X] T018 [P] `SecretStore` interface + Fernet implementation in `backend/src/app/core/secrets.py`; `secret` model in `backend/src/app/core/models_secret.py` (ciphertext only, never serialized); migration

### Vendored LORM

- [X] T019 [P] Vendor the LORM reference implementation into `backend/src/app/vendor/lorm/`: copy `lorm-policy.schema.json`, `validate_policy.py`, `SPEC.md` **unmodified** from github.com/Argyronix/lorm at a **pinned commit/tag**; record the commit SHA + tag + retrieval date in `backend/src/app/vendor/lorm/UPSTREAM.md`; commit the vendored files (Apache-2.0, add `LICENSE`); wrapper `backend/src/app/policies/schema_validator.py` calling `validate_policy.py`

### Capabilities + minimal promotion

- [X] T020 [P] Capability models in `backend/src/app/lorm/models.py`: `capability` (key, level, `l5_allowed`, `verification_tolerances` jsonb, `uncertainty_threshold`), `capability_level_event` (append-only, direction/from/to/trigger/actor) (data-model.md §7); migration
- [X] T021 Capability seed in `backend/src/app/lorm/seed.py`: create the 8 capabilities with these **exact initial levels** — `proc.inventory.observe`=L0, `proc.demand.observe`=L1, `proc.risk.diagnose`=L2, `proc.order.recommend`=L3, `proc.po.create`=L3, `proc.replenish.routine`=L4, `proc.supplier.add`=L4 (`l5_allowed=false`), `proc.payment.release`=L3 (`l5_allowed=false`); `l5_allowed=true` for the other six; `verification_tolerances = {price_pct: 10, qty_short_pct: 10, late_days: 3}` (supplier mismatch is always material); `uncertainty_threshold = 0.3`. Story tests MUST explicitly set/promote the level they need and MUST NOT rely on these defaults implicitly.
- [X] T022 Add `promotion_request` model to `backend/src/app/lorm/models.py` (capability_id, proposed_to_level, rationale, evidence jsonb, `origin` `human|ai`, `status` `pending|approved|rejected`, decided_by, decided_at) (data-model.md §7); migration. (Same file as T020 — sequence after it.)
- [X] T023 Minimal capability service `backend/src/app/lorm/capability_service.py`: read current level + `capability_level_event` history; `create_promotion_request` (human origin); `approve` → apply **exactly one** level increase + write `capability_level_event(direction=promotion, actor)`; refuse a multi-level jump; refuse a target of L5 when `l5_allowed=false` (FR-049/FR-051, I-8); `reject`. This is the **only** code path that raises `capability.level`. (AI suggestions, trust-record gating, and the L4→L5 approver ≠ policy-author rule are added in US6.)
- [X] T024 Minimal capabilities router `backend/src/app/api/routers/capabilities.py`: `GET /api/v1/capabilities`, `GET /api/v1/capabilities/{key}/history`, `POST /api/v1/capabilities/{key}/promotion-requests`, `POST /api/v1/promotion-requests/{id}/approve|reject`; permissions `capability.read`, `capability.promote`; register in `main.py`. (US6 extends the payloads.)
- [X] T025 [P] Promotion-core tests `backend/tests/integration/test_promotion_core.py`: one-level-only enforced; two-level jump rejected; `l5_allowed=false` → promotion to L5 refused; `capability.level` unchanged unless via an approved request; `capability_level_event` recorded with actor

### Audit + append-only guard

- [X] T026 [P] `audit_record` model in `backend/src/app/audit/models.py` (append-only; superset of LORM §10.3 + FR-060/FR-061) + migration; `AuditRecorder` service in `backend/src/app/audit/recorder.py` that asserts required fields (`capability`, `lorm_level`, `authorizer`, `action`, `params`, `evidence_ref`, `outcome`, `verified`, `correlation_id`) on write
- [X] T027 Append-only enforcement in `backend/alembic/versions/*_append_only_guard.py`: create a reusable PL/pgSQL `forbid_mutation()` trigger function (raises on UPDATE/DELETE) and an `attach_append_only('<table>')` Alembic helper; attach it to `audit_record` and `capability_level_event` now. Every later append-only table (`mapping_change_event`, `observation_signal`, `policy_approval`, `execution_attempt`) MUST call `attach_append_only` in its own migration. Test `backend/tests/integration/test_append_only.py` asserts UPDATE and DELETE on `audit_record` raise (G2, Principle X); application-level `AppendOnly` assertions remain as defense in depth.
- [X] T028 [P] `observability_gap` model + service in `backend/src/app/observation/observability.py` (open/close a gap by scope=source|entity|capability, reason=source_unavailable|stale_data|ai_unavailable) (data-model.md §4)

### Durable job queue + worker

- [X] T029 `job` model in `backend/src/app/jobs/models.py` + migration (kind, payload, run_at, status, attempts, max_attempts, locked_by, locked_at, last_error) (data-model.md §10)
- [X] T030 `JobQueue` interface + Postgres implementation in `backend/src/app/jobs/queue.py`: transactional `enqueue`, claim with `SELECT … FOR UPDATE SKIP LOCKED`, exponential backoff reschedule, self-re-enqueue helper for recurring jobs
- [X] T031 Worker entrypoint `backend/src/app/worker.py` (`python -m app.worker`): poll loop, handler registry, graceful shutdown, `--run-once <kind>` for tests
- [X] T032 [P] Job-queue tests `backend/tests/integration/test_job_queue.py`: SKIP LOCKED fan-out, backoff, transactional enqueue rollback, recurring re-enqueue

### AI provider seam

- [X] T033 [P] `LLMProvider` protocol + `AnthropicProvider` + `DeterministicMockProvider` in `backend/src/app/ai/provider.py` (contracts/ai-structured-output.md §5); latest Claude model id in config
- [X] T034 `generate_structured` wrapper in `backend/src/app/ai/structured.py`: schema-validate output, retry with exponential backoff, total timeout, emit `ai_unavailable` audit record + open observability gap on final failure; never place secrets in prompt/context (FR-068, FR-016a)
- [X] T035 [P] AI wrapper tests `backend/tests/integration/test_ai_structured.py`: valid parse, schema-invalid → failure path, timeout → failure path, audit + gap emitted

### Seed + frontend shell

- [X] T036 Demo seed CLI `backend/src/app/seed.py` (`python -m app.seed --demo`): one `enterprise`; the full permission catalogue (`datasource.manage`, `mapping.confirm`, `domain.read`, `recommendation.request`, `approval.act`, `policy.author`, `policy.approve`, `capability.read`, `capability.promote`, `audit.read`, `user.manage` — data-model.md §1) mapped to roles; users `admin`/`buyer`/`approver`; capability seed (T021)
- [X] T037 [P] Frontend auth + shell: `frontend/src/features/auth/` (login page; **access token in memory only via `src/lib/authToken` — never `localStorage`/`sessionStorage`**; refresh token as an httpOnly cookie set by the backend; silent renew on load / on 401 via `POST /auth/refresh`), `frontend/src/components/AppLayout.tsx` with nav for Dashboard, Risks/Recommendations, Approvals, Autopilot/Policies, Capabilities, Data Sources, Executions/Orders, Audit, Users & Roles; protected-route wrapper; `frontend/src/api/authApi.ts`
- [X] T038 [P] Frontend error surface `frontend/src/lib/errorToast.ts` + RTK Query error middleware rendering the unified error model

**Checkpoint**: login works; seeded users/roles/capabilities exist at the levels in T021;
one-level capability promotion works via `POST /capabilities/{key}/promotion-requests` +
approve; append-only tables reject UPDATE/DELETE; worker drains jobs; OpenAPI published and
frontend types generated. User stories can now begin.

---

## Phase 3: User Story 1 — Data source onboarding & L0 domain map (Priority: P1) 🎯 MVP

**Goal**: Admin connects a data source, confirms AI-suggested field mappings, and views the
canonical domain map with per-entity provenance and observability.

**Independent Test**: quickstart.md §3 — connect the demo file source, test, introspect,
confirm mappings, see the Domain Map populated; nothing is applied without confirmation
(FR-004); SC-001 (one session, no code change).

### Tests for User Story 1

- [X] T039 [P] [US1] Contract tests `backend/tests/contract/test_data_sources.py` for `/data-sources` (CRUD, `/test`, `/introspect`, `/mapping-suggestions`, `/health-history`) per contracts/rest-api.md
- [X] T040 [P] [US1] Contract tests `backend/tests/contract/test_mappings.py` for `/mappings` (create, `/confirm`, `/reject`, `/retire`, PATCH) and `/data-sources/{id}/mappings`
- [X] T041 [P] [US1] Contract tests `backend/tests/contract/test_domain_map.py` for `/domain/map` and `/domain/{entity}` (provenance + observability fields present)
- [X] T042 [P] [US1] AI output test `backend/tests/integration/test_mapping_suggestion.py`: `MappingSuggestion` schema validation, unknown `source_field_path` dropped, nothing auto-applied (contracts/ai-structured-output.md §1)
- [X] T043 [P] [US1] Integration test `backend/tests/integration/test_onboarding_journey.py`: file source → test → introspect → suggest → confirm subset → sync → domain rows created with `source_provenance`

### Implementation for User Story 1

- [X] T044 [P] [US1] Integration models in `backend/src/app/integration/models.py`: `data_source`, `source_field`, `field_mapping`, `mapping_change_event` (call `attach_append_only` for `mapping_change_event`) (data-model.md §2); migration
- [X] T045 [P] [US1] Canonical domain models in `backend/src/app/domain/models.py`: `item`, `warehouse`, `stock_level`, `supplier` (`notes text`), `item_supplier`, `price`, `lead_time`, `purchase_order`, `consumption`, `production_demand`, `quality_record` (`note text`), each with `source_provenance` jsonb + `observability` enum (data-model.md §3); migration. **No vector columns / no `pgvector` in v1** — semantic retrieval is deferred extensibility (research.md §13).
- [X] T046 [US1] `SourceConnector` protocol + models in `backend/src/app/integration/connectors/base.py` (contracts/source-connector.md)
- [X] T047 [P] [US1] `RestSourceConnector` in `backend/src/app/integration/connectors/rest.py` (paths, pagination, incremental param, auth via `SecretStore`)
- [X] T048 [P] [US1] `FileSourceConnector` in `backend/src/app/integration/connectors/file.py` (CSV/JSON upload parse)
- [X] T049 [P] [US1] `SqlSourceConnector` in `backend/src/app/integration/connectors/sql.py` (read-only DSN; reject non-`SELECT`)
- [X] T050 [US1] Data-source service `backend/src/app/integration/service.py`: create/update, `test_connection` → `data_source.health` + open/close `observability_gap`, `describe_schema` → persist `source_field`
- [X] T051 [US1] Mapping-suggestion orchestration in `backend/src/app/integration/mapping_ai.py` + `suggest_mapping` job handler: call `generate_structured(MappingSuggestionSet)`, persist each as `field_mapping(status=suggested)` (FR-003/FR-004)
- [X] T052 [US1] `FieldMapping` lifecycle service `backend/src/app/integration/mapping_service.py`: confirm/reject/retire/edit → append `mapping_change_event`; only `confirmed` mappings exposed to sync (FR-005/FR-006)
- [X] T053 [US1] Sync service `backend/src/app/integration/sync.py` + `observe_source` job handler (data path only; signal diffing added in US2): `fetch(since)` → map through confirmed `field_mapping` → upsert `domain` rows with `source_provenance` + `observability` (FR-007/FR-013)
- [X] T054 [US1] Domain-map read service `backend/src/app/domain/map_service.py`: entity/relationship summary + per-entity source origin + stale/lost flags (FR-008–FR-011)
- [X] T055 [US1] Routers: `backend/src/app/api/routers/data_sources.py` and `mappings.py` and `domain.py` (endpoints per contracts/rest-api.md); register in `main.py`; permissions `datasource.manage`, `mapping.confirm`, `domain.read`
- [X] T056 [P] [US1] Frontend `frontend/src/api/dataSourcesApi.ts` + `frontend/src/api/domainApi.ts` (RTK Query slices)
- [X] T057 [P] [US1] Frontend Data Sources feature `frontend/src/features/datasources/`: list, connect form, test, introspect, mapping-review table with confirm/edit/reject, health history
- [X] T058 [P] [US1] Frontend Domain Map feature `frontend/src/features/domain/`: entity list/detail with provenance and observability badges
- [X] T059 [US1] Seed fixture `backend/tests/fixtures/demo_enterprise.csv` (items, stock, suppliers, prices, lead times, consumption, production demand, one open PO with future `expected_at`) used by quickstart §3

**Checkpoint**: US1 fully functional and testable on its own (MVP).

**Note**: `GET /api/v1/dashboard` was added as Phase 3 polish. It exposes only the Phase-3-safe
read model — LORM `open_risks`/`recommendations`/`approvals` as `null` placeholders and a real
`autopilot` (L5-capability) count, alongside data-source/domain-map health. The full dashboard
aggregation service (`risk_finding`/`recommendation`/`procurement_action` counts) is T072/T073
below and remains part of later user stories — it is **not** considered done.

---

## Phase 4: User Story 2 — Observation & risk detection with explanation (Priority: P1)

**Goal**: The system observes signals headless, deterministically detects risks, and attaches
an AI explanation with evidence and confidence; losing inputs is made explicit.

**Independent Test**: quickstart.md §4 — inject an approaching-shortage dataset; a
`likely_shortage` risk with explanation and listed evidence appears on the Dashboard before
the modelled stock-out; no UI needed for detection (SC-002).

### Tests for User Story 2

- [ ] T060 [P] [US2] Rule tests `backend/tests/unit/test_risk_rules.py`: `likely_shortage`, `insufficient_until_next_delivery`, `production_stop_risk` (from `production_demand` vs projected cover), `systematic_supplier_delay`, `price_anomaly`, `quality_degradation` from fixture aggregates
- [ ] T061 [P] [US2] AI output test `backend/tests/integration/test_risk_explanation.py`: `RiskExplanation` schema, non-empty `data_used`, refs not linked to the finding rejected, empty → `ai_status=unavailable` (contracts/ai-structured-output.md §2)
- [ ] T062 [P] [US2] Integration test `backend/tests/integration/test_observation_loop.py`: signals ingested, `sku_aggregate` recomputed, risk created before stock-out, no duplicate risk for an open finding
- [ ] T063 [P] [US2] Observability test `backend/tests/integration/test_observability_loss.py`: source `unavailable` → gap opened, dependent autonomous output blocked (FR-015/FR-016)
- [ ] T064 [P] [US2] Contract tests `backend/tests/contract/test_risks.py` and `test_dashboard.py`

### Implementation for User Story 2

- [ ] T065 [P] [US2] Observation models in `backend/src/app/observation/models.py`: `observation_signal` (append-only, month-range partitioned — call `attach_append_only`), `sku_aggregate` (data-model.md §4); migration
- [ ] T066 [US2] Extend `observe_source` handler (`backend/src/app/integration/sync.py`): diff upserted domain rows into `observation_signal` rows (FR-014)
- [ ] T067 [US2] `recompute_aggregates` job in `backend/src/app/observation/aggregates.py`: deterministic `avg_daily_consumption`, `days_of_cover`, `next_expected_delivery_at`, `last_price`, `price_trend`, `avg_supplier_delay_days` (research.md §7a — pre-AI aggregation)
- [ ] T068 [US2] Scheduler wiring in `backend/src/app/jobs/scheduler.py`: on `observe_source` completion re-enqueue at `data_source.observation_interval`; chain `recompute_aggregates` → `detect_risks`
- [ ] T069 [P] [US2] Analysis models in `backend/src/app/analysis/models.py`: `risk_finding` (+ state machine), `risk_signal_link`, `explanation` (subject_type/subject_id) (data-model.md §5); migration
- [ ] T070 [US2] Deterministic risk rules `backend/src/app/analysis/rules.py` + `detect_risks` job: create `risk_finding` + `risk_signal_link` (evidence, I-1); dedupe against open findings. Rules MUST cover `likely_shortage`, `insufficient_until_next_delivery`, `production_stop_risk` (using `production_demand`), `systematic_supplier_delay`, `price_anomaly`, `quality_degradation` (FR-017/FR-019)
- [ ] T071 [US2] `generate_explanation` job in `backend/src/app/analysis/explain.py`: `generate_structured(RiskExplanation)` with evidence-ref validation; set `risk_finding.ai_status` (`ready`/`unavailable`) (FR-018/FR-070, I-2/I-3)
- [ ] T072 [US2] Dashboard aggregation service `backend/src/app/analysis/dashboard.py`: open risks, pending-approval count, active policies, capability levels, source health, AI-unavailable items, recent executions
- [ ] T073 [US2] Routers `backend/src/app/api/routers/risks.py` (`/risks`, `/risks/{id}`, `/risks/{id}/dismiss`) and `dashboard.py` (`/dashboard`); permissions `domain.read`
- [ ] T074 [P] [US2] Frontend `frontend/src/api/risksApi.ts` + `frontend/src/api/dashboardApi.ts`
- [ ] T075 [P] [US2] Frontend Dashboard feature `frontend/src/features/dashboard/`
- [ ] T076 [P] [US2] Frontend Risks feature `frontend/src/features/risks/`: list + detail (explanation, factors, confidence, linked evidence, "AI unavailable — needs human" state), dismiss

**Checkpoint**: US1 + US2 both work independently.

---

## Phase 5: User Story 3 — Procurement recommendation (Priority: P1)

**Goal**: For a detected risk, produce a structured recommendation (what/how much/when/which
supplier/why + expected outcome, risks, alternatives, do-nothing, confidence); at L3 it is
shown only.

**Independent Test**: quickstart.md §5 — request a recommendation for an open risk; every
mandatory element present (SC-003); `procurement_action` ends `recommendation_only`; an
`audit_record(event_type=recommendation)` exists; nothing dispatched (FR-026).

### Tests for User Story 3

- [ ] T077 [P] [US3] AI output test `backend/tests/integration/test_recommendation_schema.py`: `ProcurementRecommendation` — `proposed_supplier_ref` must be an approved supplier for the item, `quantity>0`, currency = enterprise base, `alternatives` ≥ 1 when >1 eligible supplier (contracts/ai-structured-output.md §3)
- [ ] T078 [P] [US3] Enforcement test `backend/tests/integration/test_enforcement_l3.py`: capability `proc.po.create` at its seeded L3 → `evaluate()` returns `deny`, `effective_level=L3`; low `confidence` → not eligible for autonomous (FR-025, I-3)
- [ ] T079 [P] [US3] Contract test `backend/tests/contract/test_recommendations.py` (`/risks/{id}/recommendation`, `/recommendations/{id}`)
- [ ] T080 [P] [US3] E2E-support fixture + test `backend/tests/integration/test_e2e_l3.py`: risk → recommendation → `recommendation_only` (steps 1–8 of the primary scenario)

### Implementation for User Story 3

- [ ] T081 [P] [US3] Decision models in `backend/src/app/decision/models.py`: `recommendation` (+ `explanation` subject=recommendation), `procurement_action` (+ full status state machine, `idempotency_key`, `enforcement_decision` jsonb) (data-model.md §6); migration
- [ ] T082 [US3] Recommendation builder `backend/src/app/decision/recommend.py` + `generate_recommendation` job: gather aggregates/domain context, `generate_structured(ProcurementRecommendation)`, backend validation (approved-supplier, currency, qty), build `alternatives` comparison, persist `recommendation` + `explanation`
- [ ] T083 [US3] `procurement_action` assembler `backend/src/app/decision/action_builder.py`: create `prepared` action from a recommendation with a stable `idempotency_key`
- [ ] T084 [US3] `LormEnforcementService` v1 in `backend/src/app/lorm/enforcement.py` (contracts/lorm-enforcement.md rules 1–3): I-1 evidence gate (missing/stale diagnosis → `deny`), capability level cap (L0–L3 → `deny`/L3, L4 → `ask`, L5 → placeholder `ask` until US5), uncertainty degrade; emit HMAC `decision_token`; write enforcement `audit_record`
- [ ] T085 [US3] Decision orchestration in `backend/src/app/decision/orchestrator.py`: on `risk_finding` needing action → `generate_recommendation` → `action_builder` → `enforcement.evaluate()`; `deny` → `procurement_action.status=recommendation_only` + audit (FR-021–FR-026)
- [ ] T086 [US3] Router `backend/src/app/api/routers/recommendations.py` (`POST /risks/{id}/recommendation`, `GET /recommendations/{id}`); permission `recommendation.request`
- [ ] T087 [P] [US3] Frontend `frontend/src/api/recommendationsApi.ts`
- [ ] T088 [P] [US3] Frontend Recommendations feature `frontend/src/features/recommendations/`: detail view (proposed action, expected outcome, reasons, risks, alternatives table, do-nothing consequence, confidence)

**Checkpoint**: MVP complete — US1 + US2 + US3 deliver observe → explain → recommend (L3).

---

## Phase 6: User Story 4 — Execute after human approval (L4) (Priority: P2)

**Goal**: At L4 the system prepares an action, a permitted user approves/rejects it, and on
approval it is dispatched idempotently through a swappable adapter; results and restart
recovery are handled.

**Independent Test**: quickstart.md §6 — promote `proc.po.create` L3→L4 via the Foundational
promotion endpoint, approve a prepared action → dispatched to the simulated adapter →
`executed`; reject → nothing dispatched; kill worker mid-dispatch → `reconcile_executions`
resolves without a second order (FR-032a); full audit chain via `/audit/action/{id}` (SC-004).

### Tests for User Story 4

- [ ] T089 [P] [US4] Adapter tests `backend/tests/integration/test_execution_adapters.py`: `SimulatedExecutionAdapter` idempotency + `get_status`; `GenericRestExecutionAdapter` against a stand-in HTTP server, HTTP error → unified error model (contracts/execution-adapter.md)
- [ ] T090 [P] [US4] Idempotency + recovery test `backend/tests/integration/test_execution_recovery.py`: repeat `dispatch` same key → one order; `sent_unknown` after restart → reconciled via `get_status`; undispatched durable action → re-enqueued; recovery audited (FR-032a)
- [ ] T091 [P] [US4] Approval flow test `backend/tests/integration/test_l4_approval.py`: promote `proc.po.create` L3→L4 via `POST /api/v1/capabilities/proc.po.create/promotion-requests` + approve; then prepared → approve → authorized → dispatched → executed; reject → `rejected` + reason; approver without `approval.act` → 403
- [ ] T092 [P] [US4] Contract tests `backend/tests/contract/test_approvals.py` and `test_executions.py`
- [ ] T093 [P] [US4] E2E test `backend/tests/integration/test_e2e_l4.py`: primary scenario with `proc.po.create` promoted to L4 via the Foundational promotion endpoint

### Implementation for User Story 4

- [ ] T094 [P] [US4] Execution models in `backend/src/app/execution/models.py`: `execution_attempt` (append-only — call `attach_append_only`), `simulated_order` (idempotency table) (data-model.md §8); migration
- [ ] T095 [US4] `ExecutionAdapter` protocol + command/result/status models in `backend/src/app/execution/adapters/base.py` (contracts/execution-adapter.md)
- [ ] T096 [P] [US4] `SimulatedExecutionAdapter` in `backend/src/app/execution/adapters/simulated.py` (deterministic, DB-backed idempotency, config latency/failure)
- [ ] T097 [P] [US4] `GenericRestExecutionAdapter` in `backend/src/app/execution/adapters/generic_rest.py` (config from `enterprise` + `secret`, idempotency header, HTTP→`AdapterError` mapping, `get_status`)
- [ ] T098 [US4] Execution service `backend/src/app/execution/service.py`: `dispatch(action, decision: EnforcementDecision)` guard (refuse without matching `allow` token, FR-069), adapter selection by `enterprise.active_execution_adapter`, write `execution_attempt`, update `procurement_action.status`
- [ ] T099 [US4] `dispatch_action` + `reconcile_executions` job handlers in `backend/src/app/execution/jobs.py`: reconcile `sent_unknown` via `get_status`, re-enqueue undispatched durable actions, audit every recovery step (FR-032a)
- [ ] T100 [US4] Approval workflow `backend/src/app/decision/approvals.py`: route `ask` decisions into the L4 queue; approve → `authorized` + enqueue `dispatch_action` + audit; reject → `rejected` + reason + audit (FR-027–FR-030, FR-033)
- [ ] T101 [US4] Extend `LormEnforcementService` in `backend/src/app/lorm/enforcement.py` so the L4 capability cap reliably yields `ask` and records `degraded_from` when applicable
- [ ] T102 [US4] Routers `backend/src/app/api/routers/approvals.py` (`/approvals`, `/approvals/{id}`, `/approve`, `/reject`) and `executions.py` (`/executions`, `/executions/{id}`); permission `approval.act`, `domain.read`; add `PATCH /enterprise` adapter switch handling
- [ ] T103 [P] [US4] Frontend `frontend/src/api/approvalsApi.ts` + `frontend/src/api/executionsApi.ts`
- [ ] T104 [P] [US4] Frontend Approvals feature `frontend/src/features/approvals/`: pre-approval view (item, qty, supplier, price/cost, needed-by, AI explanation, risks, data used — FR-028), approve/reject with reason
- [ ] T105 [P] [US4] Frontend Executions/Orders feature `frontend/src/features/executions/`: list + detail (attempts, status, external ref)

**Checkpoint**: US1–US4 independently functional; the system can now act with a human in the loop.

---

## Phase 7: User Story 5 — Controlled autopilot via approved policy (L5) (Priority: P2)

**Goal**: A machine-readable L5 policy (distinct author and approver) authorizes autonomous
execution strictly within its bounds; out-of-bounds actions degrade to L4; expiry/revocation
auto-demotes.

**Independent Test**: quickstart.md §7 — promote `proc.replenish.routine` L4→L5 (one step,
via the Foundational promotion endpoint), author a policy, self-approval blocked with HTTP 409
(FR-038, SC-006), approver activates it, in-bounds cycle executes autonomously with
`authorizer={policy_id,version}`, out-of-bounds cycle degrades to L4 (FR-042, SC-005),
revoke → capability L5→L4.

### Tests for User Story 5

- [ ] T106 [P] [US5] Policy schema-validation test `backend/tests/integration/test_policy_schema.py`: canonical policy serializes and passes vendored `validate_policy.py`; invalid rejected before `pending_approval` (FR-036/FR-044)
- [ ] T107 [P] [US5] Separation-of-duties test `backend/tests/integration/test_policy_sod.py`: multi-role user who authored a policy gets HTTP 409 `policy_author_equals_approver` on approve (FR-038/FR-065)
- [ ] T108 [P] [US5] Enforcement L5 tests `backend/tests/integration/test_enforcement_l5.py`: in-bounds → `allow`/L5; over `max_order_amount` → `ask`/`degraded_from=out_of_bounds`; period-aggregate exceeded (computed from audit log) → `ask`; failing `conditions[].check` → `ask`; expired/revoked/no active policy → `ask`
- [ ] T109 [P] [US5] Contract tests `backend/tests/contract/test_policies.py` (CRUD, draft-suggestion, validate, submit, approve, reject, revoke, approvals history)
- [ ] T110 [P] [US5] E2E test `backend/tests/integration/test_e2e_l5.py`: primary scenario with `proc.replenish.routine` promoted to L5, both in-bounds and out-of-bounds branches

### Implementation for User Story 5

- [ ] T111 [P] [US5] Policy models in `backend/src/app/policies/models.py`: `policy` (canonical fields + state machine), `policy_approval` (append-only — call `attach_append_only`), `policy_draft_suggestion` (data-model.md §7); migration; DB CHECK `author_id <> approved_by_id`
- [ ] T112 [US5] Policy service `backend/src/app/policies/service.py`: create (`draft`, requires `capability.l5_allowed`), `validate` (vendored validator → `schema_valid`/`schema_errors`), `submit` (`draft`→`pending_approval` only if valid), `approve`/`reject` (enforce approver ≠ author regardless of roles, write `policy_approval`), `revoke`, edit bounds → new `version` in `draft` (FR-036–FR-044)
- [ ] T113 [P] [US5] `PolicyDraft` AI assist `backend/src/app/policies/draft_ai.py` + endpoint: `generate_structured(PolicyDraft)` stored as `policy_draft_suggestion` — advisory only, never submitted/approved by AI (FR-037, I-8)
- [ ] T114 [US5] Extend `LormEnforcementService` (`backend/src/app/lorm/enforcement.py`) with L5 policy evaluation (contracts/lorm-enforcement.md rule 4): load single `active` policy, expiry/revoke check, bounds check (skus/categories/suppliers/`max_order_amount`/period aggregate from `audit_record`), `conditions[].check` with timeout → degrade to `ask` on any failure; `allow` sets `effective_level=L5`, `policy_id`/`policy_version`
- [ ] T115 [US5] `policy_expiry_scan` job in `backend/src/app/policies/jobs.py`: on `expires_at` reached or revoke → trigger one-step demotion (wired to `lorm/demotion.py`, finalized in US7) + audit
- [ ] T116 [US5] Autopilot path in `backend/src/app/decision/approvals.py`: `allow`/L5 → enqueue `dispatch_action` without human approval; `audit_record.authorizer={policy_id,version}` (FR-041); out-of-bounds → into L4 queue (FR-042)
- [ ] T117 [US5] Router `backend/src/app/api/routers/policies.py` (all endpoints per contracts/rest-api.md); permissions `policy.author`, `policy.approve`
- [ ] T118 [P] [US5] Frontend `frontend/src/api/policiesApi.ts`
- [ ] T119 [P] [US5] Frontend Autopilot/Policies feature `frontend/src/features/policies/`: structured policy editor (capability, bounds, conditions, verification, expiry), validate, submit, approve/reject/revoke, status + approvals history, draft-suggestion assist

**Checkpoint**: US1–US5 — full L3/L4/L5 decision flow works.

---

## Phase 8: User Story 6 — Manage capability autonomy levels (Priority: P2)

**Goal**: Enrich the Foundational promotion mechanism with AI suggestions, the L4→L5
approver ≠ policy-author rule, trust-record context, richer endpoints, and the Capabilities
UI. (The minimal human-only one-level promotion + `l5_allowed` guard already exist from
Foundational.)

**Independent Test**: quickstart.md §6 groundwork — list capabilities, promote one by one
step (already works), reject a two-step jump, see an AI suggestion that does not auto-apply,
confirm L5 blocked for capped capabilities (SC-007, SC-008).

### Tests for User Story 6

- [ ] T120 [P] [US6] Promotion tests `backend/tests/integration/test_promotion.py`: AI-origin `promotion_request` never auto-applied; trust-record summary present in `/capabilities/{key}/history`; extends `test_promotion_core`
- [ ] T121 [P] [US6] SoD-on-promotion test `backend/tests/integration/test_promotion_sod.py`: L4→L5 promotion blocked when approver == active policy author (SPEC §6.2)
- [ ] T122 [P] [US6] Contract tests `backend/tests/contract/test_capabilities.py` (`/capabilities`, `/history`, `/promotion-requests`, approve/reject, extended payloads)

### Implementation for User Story 6

- [ ] T123 [US6] Extend `backend/src/app/lorm/capability_service.py`: L4→L5 approval requires `decided_by` ≠ the capability's active policy author (SPEC §6.2); include a trust-record (verification history) summary in `history`; link an accepted `promotion_request` to its originating AI suggestion (FR-050/FR-051)
- [ ] T124 [US6] AI promotion suggestion `backend/src/app/lorm/promotion_ai.py`: create `promotion_request(origin=ai)` from the accumulated trust record; never auto-applies (FR-052/FR-053, I-8)
- [ ] T125 [US6] Extend `backend/src/app/api/routers/capabilities.py`: promotion-request list/detail with history, AI-suggestion surfacing on the capability payload
- [ ] T126 [P] [US6] Frontend `frontend/src/api/capabilitiesApi.ts`
- [ ] T127 [P] [US6] Frontend Capabilities feature `frontend/src/features/capabilities/`: level per capability, `l5_allowed`, tolerances, active policy link, level-change history, promotion-request list + approve/reject

**Checkpoint**: US1–US6 — autonomy levels are managed under human control.

---

## Phase 9: User Story 7 — Verification & automatic demotion (Priority: P3)

**Goal**: L4/L5 outcomes are verified against configurable tolerances; material failure,
policy violation, observability loss, uncertainty, or policy expiry auto-demotes the
capability one level with no human/AI approval.

**Independent Test**: quickstart.md §8 — feed a "received" fixture 25% over expected price
(exceeds the 10% default) → `verification_result=failed` → capability demoted one level,
recorded with reason/time/levels (SC-009); observability-loss and AI-outage also demote/block
(SC-010).

### Tests for User Story 7

- [ ] T128 [P] [US7] Tolerance evaluation tests `backend/tests/unit/test_verification_tolerances.py`: price % (default 10), qty short % (default 10), late days (default 3), supplier mismatch = always material; capability defaults vs policy override (FR-055a)
- [ ] T129 [P] [US7] Demotion tests `backend/tests/integration/test_demotion.py`: each §6.3 trigger (verification_failure, incident/rollback, observability_loss, uncertainty, policy_expiry, policy_revocation) → exactly one level down, no approval, `capability_level_event` written (FR-056/FR-057)
- [ ] T130 [P] [US7] Unverifiable test `backend/tests/integration/test_unverifiable.py`: window elapsed with no data → `unverifiable`; repeated `unverifiable` → surfaced, blocks L5 eligibility (SPEC §10.1)
- [ ] T131 [P] [US7] Contract test `backend/tests/contract/test_verification.py` (`/executions/{id}/verification`)

### Implementation for User Story 7

- [ ] T132 [P] [US7] `verification_result` model in `backend/src/app/verification/models.py` (+ state machine, measurements jsonb, material flags) (data-model.md §8); migration
- [ ] T133 [US7] Tolerance evaluator `backend/src/app/verification/tolerances.py`: resolve capability defaults (`price_pct=10`, `qty_short_pct=10`, `late_days=3`, supplier mismatch always material) + optional policy override, compute per-dimension deviation + `material` flag (FR-055a)
- [ ] T134 [US7] `run_verification` job in `backend/src/app/verification/service.py`: schedule at dispatch with a `verification_window`; compare observed domain data vs expected; set `verified|failed|unverifiable|pending`; on window elapsed → `unverifiable`
- [ ] T135 [US7] `lorm/demotion.py` in `backend/src/app/lorm/demotion.py`: decrement `capability.level` by exactly one + write `capability_level_event(direction=demotion, trigger=...)` transactionally, no approval; idempotent per trigger event
- [ ] T136 [US7] Wire demotion triggers: `verification/` (`failed` + `overall_material` + not `explained_accepted_by`), `analysis/` (observability_gap for capability inputs incl. `ai_unavailable`; sustained uncertainty), `execution/` (adapter-reported incident / invoked reversal), `policies/policy_expiry_scan` (FR-056)
- [ ] T137 [US7] Extend `LormEnforcementService` in `backend/src/app/lorm/enforcement.py`: block L5 eligibility when a capability has repeated `unverifiable` results
- [ ] T138 [US7] Router additions `backend/src/app/api/routers/executions.py`: `GET /executions/{id}/verification`; extend `/capabilities/{key}/history` payload with demotion reason/from/to
- [ ] T139 [P] [US7] Frontend: Executions verification detail panel in `frontend/src/features/executions/`; demotion entries in `frontend/src/features/capabilities/` history

**Checkpoint**: US1–US7 — the responsibility loop closes; autonomy self-corrects.

---

## Phase 10: User Story 8 — Audit trail of decisions and authority (Priority: P3)

**Goal**: Every significant decision/action is reconstructable — proposal, evidence,
capability, LORM level, authorization source, execution result, verification, level changes,
failures/recovery — via the UI, independent of LLM context.

**Independent Test**: quickstart.md §9 — run an action through L4 and L5, find both in Audit
with all §10.3/FR-060 fields; `/audit/action/{id}` shows the full chain; restart the DB and
clear LLM context — all records persist (SC-011, FR-063).

### Tests for User Story 8

- [ ] T140 [P] [US8] Audit completeness test `backend/tests/integration/test_audit_chain.py`: L4 and L5 actions each produce a record with `capability`, `lorm_level`, `authorizer`, `action`, `params`, `evidence_ref`, `outcome`, `verified`; `/audit/action/{id}` returns recommendation→enforcement→approval/policy→dispatch→result→verification→demotion
- [ ] T141 [P] [US8] Persistence test `backend/tests/integration/test_audit_persistence.py`: records, capability levels, policies, confirmed mappings all survive a fresh DB session (FR-063)
- [ ] T142 [P] [US8] Contract test `backend/tests/contract/test_audit.py` (`/audit`, `/audit/{id}`, `/audit/action/{id}`, filters)

### Implementation for User Story 8

- [ ] T143 [US8] Audit query service `backend/src/app/audit/query.py`: filter by capability / event_type / level / date range / correlation_id; assemble per-action chain
- [ ] T144 [US8] Cross-cutting emission audit: verify/add `AuditRecorder` calls for `mapping_confirmed`, `policy_approved`, `policy_revoked`, `promotion`, `demotion`, `ai_unavailable`, `recovery` across `integration/`, `policies/`, `lorm/`, `ai/`, `execution/` (FR-059)
- [ ] T145 [US8] Router `backend/src/app/api/routers/audit.py` (`/audit`, `/audit/{id}`, `/audit/action/{procurement_action_id}`); permission `audit.read`
- [ ] T146 [P] [US8] Frontend `frontend/src/api/auditApi.ts`
- [ ] T147 [P] [US8] Frontend Audit feature `frontend/src/features/audit/`: filterable list, record detail, action-chain view

**Checkpoint**: US1–US8 — the system is fully auditable.

---

## Phase 11: User Story 9 — Users & roles with LORM separation of duties (Priority: P3)

**Goal**: Admin manages users, roles, and multi-role assignment; LORM constraints (author ≠
approver) hold regardless of roles; backend enforces every permission, frontend is untrusted.

**Independent Test**: quickstart.md §9 groundwork — assign a user both Buyer and Approver;
they still cannot approve their own policy; a permission-less direct API call is rejected by
the backend (SC-013).

### Tests for User Story 9

- [ ] T148 [P] [US9] Multi-role SoD test `backend/tests/integration/test_multirole_sod.py`: Buyer+Approver user blocked from approving own policy and own L4→L5 promotion
- [ ] T149 [P] [US9] Backend-boundary test `backend/tests/integration/test_authz_boundary.py`: state-changing endpoints reject calls lacking the permission even with a valid token; LORM enforcement not bypassable (FR-066/FR-067/FR-069)
- [ ] T150 [P] [US9] Contract tests `backend/tests/contract/test_users.py` (`/users`, `/users/{id}`, `/users/{id}/roles`, `/roles`, `/permissions`)

### Implementation for User Story 9

- [ ] T151 [US9] User management service `backend/src/app/identity/user_service.py`: create/update/deactivate user, assign multiple roles, read role + permission catalogue
- [ ] T152 [US9] Router `backend/src/app/api/routers/users.py` (`/users` CRUD, `/users/{id}/roles`, `/roles`, `/permissions`); permission `user.manage`
- [ ] T153 [P] [US9] Frontend `frontend/src/api/usersApi.ts`
- [ ] T154 [P] [US9] Frontend Users & Roles feature `frontend/src/features/users/`: user list/create/edit, multi-role assignment, permission catalogue view

**Checkpoint**: all nine user stories independently functional.

---

## Phase 12: Polish & Cross-Cutting Concerns

**Purpose**: End-to-end validation, non-functional targets, documentation.

- [ ] T155 [P] Playwright E2E `frontend/e2e/primary-scenario.spec.ts`: the 14-step primary scenario run three times (capability at L3, L4, L5 — L4/L5 promoted via the promotion endpoint) with mock provider + simulated adapter; assert a consistent audit chain each time (SC-012)
- [ ] T156 [P] Adapter-swap test `backend/tests/integration/test_adapter_swap.py`: the same recommendation set executes via `simulated` then `generic_rest` with no change to `decision/`, `lorm/`, `verification/` (SC-014)
- [ ] T157 [P] Extensibility test `backend/tests/integration/test_extensibility.py`: add a trivial new capability row and a new `file` source without touching core modules (SC-015)
- [ ] T158 [P] Load test `load/observation.js` (k6) or `load/locustfile.py`: ~100k `observation_signal`/day, 10k SKUs, 10 sources against simulated adapter + mock provider; assert no growing `job` backlog and risks surface within one observation interval (SC-016)
- [ ] T159 [P] `docs/architecture.md` — layer/module map, boundary rules, worker process
- [ ] T160 [P] `docs/lorm-integration.md` — reuse / adapt / build split (from research.md §6 + contracts/lorm-enforcement.md), pinned upstream LORM commit/tag, demotion vs promotion semantics. **Document the v1/demo simplification**: L4→L5 promotion enforces human-only + one-level + `l5_allowed`, but does **not** gate on the SPEC §6.2 RECOMMENDED ≥10-execution L4 track record (RECOMMENDED, not MUST). US6 surfaces the trust record for the approver; a hard threshold is deliberately not added in v1.
- [ ] T161 [P] `docs/api-integration.md` — OpenAPI usage, unified error model, connector/adapter contracts
- [ ] T162 [P] `docs/development-setup.md` + `docs/repository-map.md` — three-repo layout, local run, env vars, seed
- [ ] T163 [P] `docs/adr/` — ADRs for: Postgres-backed job queue vs broker; in-process LORM enforcement adapter; JWT+RBAC; modular monolith; minimal promotion in Foundational
- [ ] T164 [P] Update root `README.md` run instructions with the finalized commands; fill frontend/backend repo URLs
- [ ] T165 Security hardening: login rate-limiting in `backend/src/app/api/routers/auth.py`, secret redaction filter in `backend/src/app/core/logging.py`, and a test `backend/tests/integration/test_secret_redaction.py` asserting no secret is serialized by any response model (FR-068, §15)
- [ ] T166 List-endpoint pagination/sort defaults review across all routers in `backend/src/app/api/routers/` (cursor pagination consistency)
- [ ] T167 Run `quickstart.md` end to end against a fresh environment; fix any gaps; record results in `docs/development-setup.md`

---

## Dependencies & Execution Order

### Phase dependencies

- **Setup (Phase 1)**: no dependencies.
- **Foundational (Phase 2)**: depends on Setup. **Blocks all user stories.** Now also
  contains the *minimal* human-only one-level capability promotion (T022–T025) and the
  append-only DB guard (T027).
- **User Stories (Phases 3–11)**: each depends only on Foundational. Recommended order is
  priority order; P1 stories (US1→US2→US3) also form a data/decision chain and are best done
  in sequence. US4/US5 depend on US3's `procurement_action` + enforcement. **US6 no longer
  blocks US4/US5** — it enriches the promotion mechanism that already exists in Foundational.
  US7 wires demotion triggers added across US2/US4/US5. US8 audits all. US9 extends
  Foundational identity.
- **Polish (Phase 12)**: depends on the user stories it exercises (US1–US7 for E2E; all for docs).

### Story-level dependencies

| Story | Hard prereq | Soft integration |
|-------|-------------|------------------|
| US1 | Foundational | — |
| US2 | Foundational | consumes US1 domain rows + `production_demand` |
| US3 | Foundational | consumes US2 risks + US1 domain; builds `LormEnforcementService` v1 |
| US4 | Foundational (incl. minimal promotion), US3 | — |
| US5 | Foundational (incl. minimal promotion), US3, US4 dispatch path | — |
| US6 | Foundational | enriches US4/US5 capability UX; blocks none of their independent tests |
| US7 | Foundational, US4/US5 (things to verify) | demotion triggers wired from US2/US4/US5 |
| US8 | Foundational | richer once US3–US7 emit events |
| US9 | Foundational (identity models) | cross-checks US5/US6 SoD |

### Within each story

Tests → models → services → jobs/enforcement → routers → frontend API slice → frontend feature.

### Parallel opportunities

- Setup: T002, T003, T004, T007, T008, T009, T010 in parallel after T001.
- Foundational: identity models (T011), enterprise (T017), secrets (T018), LORM vendoring
  (T019), capability models (T020), audit model (T026), observability (T028), LLM provider
  (T033) can start in parallel. Then: auth chain T012–T016; capability seed T021 (after T020);
  **promotion infra T022→T025 is sequential and depends on T020**; append-only guard T027
  depends on T020 + T026; job queue T029→T032; AI wrapper T034/T035; seed + frontend shell
  T036–T038 last.
- Within a story: all `[P]` model tasks together; all `[P]` test tasks together; frontend
  `[P]` feature tasks run alongside backend once their API slice exists.
- Across stories: once Foundational is done, US1 and US9 can proceed fully in parallel; US6
  can start immediately (it only enriches Foundational's promotion path); US2 starts as soon
  as US1 models land.

---

## Parallel Example: User Story 1

```bash
# Tests for US1 together:
Task: "Contract tests backend/tests/contract/test_data_sources.py"
Task: "Contract tests backend/tests/contract/test_mappings.py"
Task: "Contract tests backend/tests/contract/test_domain_map.py"
Task: "AI output test backend/tests/integration/test_mapping_suggestion.py"

# Models for US1 together:
Task: "Integration models in backend/src/app/integration/models.py"
Task: "Canonical domain models in backend/src/app/domain/models.py"

# Connectors for US1 together:
Task: "RestSourceConnector in backend/src/app/integration/connectors/rest.py"
Task: "FileSourceConnector in backend/src/app/integration/connectors/file.py"
Task: "SqlSourceConnector in backend/src/app/integration/connectors/sql.py"
```

---

## Implementation Strategy

### MVP (Phases 1–5)

Setup → Foundational → US1 → US2 → US3. Delivers the L0–L3 slice: connect data, observe,
detect and explain risk, produce a recommendation. **Stop and validate** with quickstart
§§3–5 (SC-001, SC-002, SC-003) before proceeding.

### Incremental delivery

1. MVP (above) → demo.
2. + US4 → human-approved execution + restart recovery → demo (SC-004, FR-032a).
3. + US5 → controlled autopilot → demo (SC-005, SC-006).
4. + US6 → enriched autonomy management (can be slotted any time after Foundational).
5. + US7 → verification & auto-demotion (SC-009, SC-010).
6. + US8 → full audit (SC-011).
7. + US9 → user/role administration.
8. Polish → E2E (SC-012), swappability (SC-014/SC-015), load (SC-016), docs.

### Parallel team strategy

After Foundational: Dev A drives the P1 chain US1→US2→US3; Dev B builds US9 + the execution
adapters/models needed by US4; Dev C builds the frontend shell features per story as API
slices land. US6 can start right after Foundational; US7/US8 after US4/US5.

---

## Notes

- `[P]` = different files, no incomplete dependency.
- Every state-changing path must call `LormEnforcementService.evaluate()` and pass the
  `decision_token` to `execution.dispatch()` — no other route to an adapter (FR-069).
- `capability.level` is raised only by `lorm/capability_service.py` (human, one level;
  minimal version in Foundational, enriched in US6); lowered only by `lorm/demotion.py`
  (automatic, one level).
- Append-only tables (`audit_record`, `capability_level_event`, `mapping_change_event`,
  `observation_signal`, `policy_approval`, `execution_attempt`) are protected by the
  `forbid_mutation()` DB trigger — never issue UPDATE/DELETE against them.
- Capability seed levels are fixed in T021; story tests must promote/set the level they need
  explicitly, never relying on the shared seed implicitly.
- LLM output is always schema-validated; invalid == unavailable → fail-safe path (FR-016a).
- Frontend user-facing strings should be routed through a single `frontend/src/lib/i18n/`
  module (English only in v1) to stay i18n-ready; bilingual resources are out of v1 scope.
- Commit after each task or logical group; keep `spec.md` / `plan.md` / `data-model.md` /
  `contracts/` in sync when a task reveals a gap, then re-run `/speckit-analyze`.
