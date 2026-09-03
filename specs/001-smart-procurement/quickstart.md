# Quickstart — Smart Procurement validation guide

A runnable path that proves the feature end to end. It references
[`data-model.md`](./data-model.md) and [`contracts/`](./contracts/) rather than repeating
schemas. No implementation code here.

## 0. Prerequisites

- PostgreSQL 16 (plain; `pgvector` is **not** required in v1).
- Python 3.12, Node 20.
- Three repos checked out into one workspace:
  ```
  SmartProcurement/            (root repo)
  ├── backend/                 (SmartProcurement-Backend)
  └── frontend/                (SmartProcurement-Frontend)
  ```
  `backend/` and `frontend/` are in the root `.gitignore`.

## 1. Configure

`backend/.env` (git-ignored):
```
DATABASE_URL=postgresql+asyncpg://sp:sp@localhost:5432/smart_procurement
JWT_SECRET=dev-only-change-me
FERNET_KEY=<generate: python -c "from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())">
LLM_PROVIDER=mock            # 'mock' for the deterministic suite; 'anthropic' for real
ANTHROPIC_API_KEY=           # only when LLM_PROVIDER=anthropic
```

## 2. Initialize

```
cd backend
uv sync                       # or: pip install -e .[dev]
alembic upgrade head
python -m app.seed --demo     # seeds: 1 enterprise, 3 roles, 3 users (admin/buyer/approver),
                              #        8 capabilities at fixed seed levels (tasks.md T021):
                              #          inventory.observe=L0  demand.observe=L1  risk.diagnose=L2
                              #          order.recommend=L3    po.create=L3      replenish.routine=L4
                              #          supplier.add=L4 (l5_allowed=false)  payment.release=L3 (l5_allowed=false)
                              #        verification tolerances: price 10% / qty 10% / late 3d
                              #          (supplier mismatch always material)
```

Run the two backend processes and the frontend:
```
uvicorn app.main:app --reload            # API  → http://localhost:8000  (OpenAPI at /openapi.json)
python -m app.worker                     # scheduler + durable job queue
cd ../frontend && npm i && npm run dev   # SPA  → http://localhost:5173
```

## 3. Connect a demo data source (L0) — validates US1, FR-001..FR-011

1. Log in as **admin**. Data Sources → **Connect** a `file` source; upload
   `backend/tests/fixtures/demo_enterprise.csv` (items, stock, suppliers, prices, lead times,
   consumption, production demand, one open PO with a future `expected_at`).
2. **Test** → `health: available`.
3. **Introspect** → `source_field[]` listed.
4. **Mapping suggestions** → AI (mock) returns `MappingSuggestion[]`
   (contract: `ai-structured-output.md` §1). Each appears as `field_mapping(status=suggested)`.
5. **Confirm** the mappings for sku / quantity / supplier / unit_price / lead_time /
   consumption. **Expected**: only confirmed mappings persist; a `mapping_change_event` is
   written per confirm.
6. Domain Map → entities populated, each showing `source_provenance` and `observability=fresh`.

**Pass when**: SC-001 (one session, no code change) and FR-004 (nothing applied without
confirm) hold.

## 4. Observe & detect a risk (L1→L2) — validates US2, FR-012..FR-020

1. The `observe_source` job runs at the source interval (or trigger it:
   `python -m app.worker --run-once observe_source --source <id>`).
2. It writes `observation_signal` rows, upserts `sku_aggregate`
   (`days_of_cover`, `avg_daily_consumption`, `next_expected_delivery_at`), then
   `detect_risks` creates a `risk_finding(risk_type=likely_shortage)` for the demo SKU whose
   `days_of_cover < lead_time` before the next delivery.
3. `generate_explanation` (mock provider) attaches an `Explanation`
   (contract §2) with non-empty `data_used` and a `confidence`.

**Pass when**: risk appears on the Dashboard **before** the modelled stock-out date;
the explanation lists the signals it used (FR-019); no UI was required for detection (FR-012).

## 5. L3 — recommendation only — validates US3, FR-021..FR-026

1. `proc.po.create` is at **L3** by the seed (T021) — no promotion needed here.
2. As **buyer**: open the risk → **Request recommendation**.
3. `generate_recommendation` produces a `ProcurementRecommendation` (contract §3):
   what / qty / when / supplier / why + expected outcome, risks, alternatives,
   do-nothing consequence, confidence.
4. A `procurement_action` is built with `status = recommendation_only`
   (`LormEnforcementService.evaluate()` → `deny`, `effective_level=L3`).

**Pass when**: recommendation carries every mandatory element (SC-003); nothing is dispatched
(FR-026); an `audit_record(event_type=recommendation)` exists.

## 6. L4 — approve then execute — validates US4, FR-027..FR-033, FR-032a

1. Promote `proc.po.create` **L3→L4** via the Foundational promotion endpoint:
   `POST /api/v1/capabilities/proc.po.create/promotion-requests` (as **buyer**), then
   `POST /api/v1/promotion-requests/{id}/approve` (as **approver**). One level only — a
   request that jumps to L5 is rejected. (The richer Capabilities screen arrives with US6;
   this step works via the API from the start.)
2. New risk cycle → a `procurement_action(status=prepared)` lands in **Approvals**.
3. Open it as **buyer/approver**: the pre-approval view shows item, qty, supplier,
   price/cost, needed-by, AI explanation, risks, data used (FR-028).
4. **Approve** → `authorized` → `dispatch_action` job → `SimulatedExecutionAdapter`
   (enterprise default) → `sent` → `executed`; `external_ref` stored; visible in
   Executions/Orders.
5. Repeat with **Reject** → `rejected`, reason stored, nothing dispatched.
6. **Restart-recovery check**: set the simulated adapter `latency_ms` high, approve an action,
   kill the worker mid-dispatch, restart it. **Expected**: `reconcile_executions` moves the
   action from `sent_unknown` via `adapter.get_status()` to `executed` — **no second order**;
   an `audit_record(event_type=recovery)` is written (FR-032a).

**Pass when**: SC-004 (complete audit chain via `/audit/action/{id}`), FR-032a recovery, and
role-gating of Approve/Reject all hold.

## 7. L5 — controlled autopilot — validates US5, FR-034..FR-045

1. Promote `proc.replenish.routine` **L4→L5** (one step — it is seeded at L4) via the
   promotion endpoint, approved by **approver**. Then create an L5 **policy**
   (Autopilot / Policies) as **buyer** (author): capability `proc.replenish.routine`,
   `bounds` = { demo SKUs, `max_order_amount` = 5000, `period_aggregate` = {window:"30d",
   max_amount:40000}, `suppliers` = [approved demo supplier] }, `verification.window` = "14d",
   `expires_at` = +90d.
2. **Validate** → runs vendored `validate_policy.py` → `schema_valid: true`.
3. **Submit** → `pending_approval`. Try to **approve as the same user (author)** →
   **HTTP 409 `policy_author_equals_approver`** (FR-038).
4. **Approve as `approver`** → `active`.
5. Trigger a replenishment cycle for an in-bounds SKU/qty → `LormEnforcementService.evaluate()`
   → `allow`, `effective_level=L5` → dispatched **without** human approval →
   `audit_record.authorizer = {policy_id, version}`.
6. Trigger a cycle that **exceeds** `max_order_amount` → decision `ask`,
   `degraded_from=out_of_bounds` → the action appears in **Approvals** (L4), not executed as
   L5 (FR-042).
7. **Revoke** the policy → capability auto-demotes L5→L4; a later in-bounds cycle now needs
   human approval.

**Pass when**: SC-005 (0 out-of-bounds autonomous executions), SC-006 (author≠approver
enforced regardless of roles), FR-041/FR-042/FR-043 hold.

## 8. Verification & automatic demotion — validates US7, FR-054..FR-058

1. For an executed L5 action, feed a "goods received" fixture with a **unit price 25 % above**
   expected — beyond the default `price_pct` tolerance of 10 % (T021).
2. `run_verification` writes a `verification_result(status=failed, overall_material=true)`.
3. `lorm/demotion.py` writes `capability_level_event(direction=demotion,
   trigger=verification_failure)` and drops `proc.replenish.routine` **one** level — no human
   approval, no AI approval (FR-057).
4. Capabilities → history shows the demotion with reason, time, from/to levels.
5. **Observability-loss check**: mark the data source `unavailable`; confirm dependent
   autonomous actions are blocked and (if it feeds a capability's inputs) a demotion fires.
6. **AI-outage check**: set `LLM_PROVIDER=mock` with a forced-failure fixture; a new risk's
   `ai_status` becomes `unavailable`, it shows on the Dashboard as "human-required", dependent
   L5 is blocked, an `audit_record(event_type=ai_unavailable)` is written (FR-016a).

**Pass when**: SC-009, SC-010 hold.

## 9. Audit review — validates US8, FR-059..FR-063

- Audit screen: filter by capability / event_type / level / date.
- `/audit/action/{procurement_action_id}` returns the full chain: recommendation → enforcement
  decision → approval or policy → dispatch → result → verification → any demotion.
- Restart Postgres and clear any LLM context: **all** audit rows, capability levels, policies
  and confirmed mappings are still present (SC-011, FR-063).

## 10. Primary end-to-end scenario (Playwright) — SC-012

`frontend/e2e/primary-scenario.spec.ts` runs the 14-step demo (spec §"Primary End-to-End
Scenario") three times — capability at L3, at L4, at L5 — with the mock provider and simulated
adapter. **Pass when** all three complete and each leaves a consistent audit chain.

## 11. Load test — SC-016

`k6 run load/observation.js` (or Locust): synthesize ~100 000 `observation_signal`/day for
10 000 SKUs across 10 sources against the simulated adapter + mock provider.
**Pass when**: the `job` table shows no growing backlog, `sku_aggregate` stays current, and
newly injected shortage conditions surface on the Dashboard within one observation interval.

---

### Traceability summary

| Steps | User stories | Key SC |
|---|---|---|
| 3 | US1 | SC-001 |
| 4 | US2 | SC-002, SC-010 |
| 5 | US3 | SC-003 |
| 6 | US4 | SC-004, FR-032a |
| 7 | US5 | SC-005, SC-006 |
| 8 | US7 | SC-009, SC-010 |
| 9 | US8 | SC-011 |
| 10 | US1–US7 (E2E) | SC-012, SC-013 |
| 11 | — | SC-016 |
| 6-7 (adapter swap variant) | US4/US5 | SC-014, SC-015 |
