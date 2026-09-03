# Phase 1 Data Model — Smart Procurement

Relational model for PostgreSQL 16 (SQLAlchemy 2.0 async + Alembic). Conventions:

- Every table has `id uuid pk`, `created_at`, `updated_at`.
- Every non-global table has `enterprise_id uuid not null references enterprise(id)`
  (multi-enterprise-ready; v1 deployment holds exactly one).
- `jsonb` is used for open/evolving structures (bounds, evidence, tolerances, payloads).
- Append-only tables (`audit_record`, `*_event`) have no `UPDATE`/`DELETE` grants for the app
  role; only `INSERT`/`SELECT`.
- Enum-like fields are stored as `text` with a CHECK constraint (portable, easy to extend).

Module ownership is shown per group; cross-module access is via service interfaces, not
foreign SELECTs (see research.md §2).

---

## 1. Identity & enterprise (`identity/`, `enterprise/`)

### `enterprise`
Global config boundary. `name`, `base_currency` (ISO 4217), `settings jsonb`,
`active_execution_adapter` (text: `simulated` | `generic_rest`).

### `user`
`enterprise_id`, `email` (unique per enterprise), `full_name`, `password_hash` (Argon2id),
`is_active`. Never exposes `password_hash` via API.

### `role`
`enterprise_id`, `key` (`administrator` | `buyer` | `approver`), `name`.
`buyer` is the **canonical internal role name**; it corresponds to "Procurement Specialist /
Buyer" in product terminology. `approver` corresponds to "Manager / Authorized Approver".

### `permission`
Global catalogue: `key` (e.g. `datasource.manage`, `mapping.confirm`, `approval.act`,
`policy.author`, `policy.approve`, `capability.promote`, `audit.read`, `user.manage`), `desc`.

### `role_permission` (M:N) — `role_id`, `permission_id`.

### `user_role` (M:N) — `user_id`, `role_id`.
A user MAY hold multiple roles (FR-065). LORM separation-of-duties is enforced in service
logic independent of roles (FR-038).

### `refresh_token`
`user_id`, `token_hash`, `expires_at`, `revoked_at`.

---

## 2. Data sources & mapping (`integration/`)

### `data_source`
`enterprise_id`, `name`, `kind` (`erp`|`mes`|`wms`|`db`|`api`|`file`|`other`),
`connector_type` (`rest`|`file`|`sql`), `config jsonb` (non-secret),
`credential_ref` (FK → `secret`, nullable), `observation_interval_seconds` (default 900),
`health` (`available`|`unavailable`|`stale`), `last_success_at`, `last_check_at`,
`last_error text`.

**State (`health`)**: `available` → `stale` (no successful fetch within N× interval) →
`unavailable` (test_connection fails). Recovers to `available` on a successful fetch.

### `secret`
`enterprise_id`, `purpose` (`source_credential`|`execution_endpoint`),
`ciphertext bytea` (Fernet), `created_by`. No plaintext column; not returned by any API.

### `source_field`
Discovered by `describe_schema()`. `data_source_id`, `path` (e.g. `items[].sku`),
`sample_values jsonb`, `inferred_type`.

### `field_mapping`
`data_source_id`, `source_field_path`, `canonical_entity` (text: `item`|`warehouse`|
`stock_level`|`supplier`|`item_supplier`|`price`|`lead_time`|`purchase_order`|`consumption`|
`production_demand`|`quality_record`), `canonical_attribute`, `transform jsonb` (optional),
`status` (`suggested`|`confirmed`|`rejected`|`retired`), `ai_confidence numeric`,
`confirmed_by` (FK user, null unless `confirmed`), `confirmed_at`.
Only `confirmed` mappings are used by sync (FR-004/FR-005). Edits/retire logged via
`mapping_change_event`.

### `mapping_change_event` (append-only)
`field_mapping_id`, `action` (`suggested`|`confirmed`|`edited`|`rejected`|`retired`),
`actor_id`, `before jsonb`, `after jsonb`, `at`.

---

## 3. Canonical procurement domain — L0 map (`domain/`)

Each row carries `source_provenance jsonb` (`{data_source_id, source_field_path, fetched_at}`)
and `observability` (`fresh`|`stale`|`lost`) so the L0 map can show origin and gaps
(FR-010/FR-011).

**v1 has no vector columns and does not require `pgvector`.** Semantic retrieval over
unstructured supplier/quality/document text is a planned extensibility option for a future
feature (research.md §13); v1 stores such text as plain `text` only.

### `item` — `sku` (unique per enterprise), `name`, `category`, `unit`, `is_active`.
### `warehouse` — `code`, `name`, `location`.
### `stock_level` — `item_id`, `warehouse_id`, `quantity numeric`, `min_quantity numeric`, `as_of`.
### `supplier` — `code`, `name`, `is_approved bool`, `notes text`.
### `item_supplier` — `item_id`, `supplier_id`, `preferred bool`. (approved supplier list for an item)
### `price` — `item_id`, `supplier_id`, `unit_price numeric`, `currency`, `valid_from`, `valid_to`.
### `lead_time` — `item_id`, `supplier_id`, `days numeric`, `as_of`. (observed / quoted)
### `purchase_order` — external reference model: `external_ref`, `item_id`, `supplier_id`,
`quantity numeric`, `unit_price numeric`, `status` (`open`|`received`|`cancelled`),
`ordered_at`, `expected_at`, `received_at`, `received_quantity numeric`,
`origin` (`external`|`smart_procurement`), `procurement_action_id` (nullable FK).
### `consumption` — `item_id`, `warehouse_id`, `quantity numeric`, `period_start`, `period_end`.
### `production_demand` — `item_id`, `quantity numeric`, `need_by`, `source_ref`. (optional; drives `production_stop_risk`)
### `quality_record` — `item_id`, `supplier_id`, `defect_rate numeric`, `note text`, `as_of`.

---

## 4. Observation — L1 (`observation/`)

### `observation_signal` (append-only, high volume — target ≤100 k/day)
`enterprise_id`, `data_source_id`, `signal_type` (`stock_change`|`consumption_rate`|
`reorder_point_near`|`demand_change`|`supplier_delay`|`lead_time_change`|`price_change`|
`quality_issue`|`other`), `item_id` (nullable), `supplier_id` (nullable),
`payload jsonb`, `observed_at`, `ingested_at`.
Partitioned by month (range on `observed_at`) for retention/pruning.

### `sku_aggregate` (derived, one row per item[/warehouse], upserted by the observation job)
`item_id`, `warehouse_id` (nullable), `avg_daily_consumption numeric`,
`days_of_cover numeric`, `next_expected_delivery_at`, `last_price numeric`,
`price_trend numeric`, `avg_supplier_delay_days numeric`, `computed_at`.
Deterministic; this is the pre-AI aggregation guard (research.md §7a).

### `observability_gap` (mutable lifecycle — `closed_at` is set when the gap clears)
`enterprise_id`, `scope` (`source`|`entity`|`capability`), `scope_ref`, `reason`
(`source_unavailable`|`stale_data`|`ai_unavailable`), `opened_at`, `closed_at` (nullable).
Feeds demotion (FR-015/FR-016a/§6.3).

---

## 5. Analysis — L2 (`analysis/`)

### `risk_finding`
`enterprise_id`, `risk_type` (`likely_shortage`|`production_stop_risk`|
`insufficient_until_next_delivery`|`systematic_supplier_delay`|`quality_degradation`|
`price_anomaly`), `item_id` (nullable), `supplier_id` (nullable), `severity`
(`low`|`med`|`high`), `status` (`open`|`recommended`|`actioned`|`resolved`|`dismissed`),
`detected_by` (`rule`), `ai_status` (`pending`|`ready`|`unavailable`), `detected_at`,
`resolved_at`.

**State**: `open` → `recommended` (a `recommendation` was produced) → `actioned`
(a `procurement_action` created) → `resolved` (stock restored / risk gone) | `dismissed`
(human). `ai_status`: `pending` → `ready` | `unavailable` (FR-016a).

### `risk_signal_link` (M:N) — `risk_finding_id`, `observation_signal_id`. (evidence, I-1)

### `explanation`
`subject_type` (`risk_finding`|`recommendation`), `subject_id`,
`what text`, `why text`, `data_used jsonb` (references to signals/aggregates/domain rows),
`factors jsonb`, `confidence numeric` (0–1), `generated_by` (`ai`|`rule`),
`llm_provider`, `llm_model`, `created_at`. (I-2, I-3, FR-018/FR-070)

---

## 6. Decision — L3 (`decision/`)

### `recommendation`
`enterprise_id`, `risk_finding_id`, `item_id`, `proposed_supplier_id`,
`quantity numeric`, `needed_by date`, `expected_unit_price numeric`, `currency`,
`expected_outcome text`, `reasons jsonb`, `risks jsonb`, `alternatives jsonb`
(list of `{supplier_id, quantity, unit_price, lead_time_days, why_not}`),
`do_nothing_consequence text`, `confidence numeric` (0–1),
`status` (`draft`|`presented`|`superseded`), `llm_provider`, `llm_model`, `created_at`.
An `explanation` row (subject = recommendation) accompanies it (FR-023/FR-070).
`confidence` below the configured escalation threshold ⇒ never eligible for autonomous
execution (FR-025, I-3).

### `procurement_action`
The ERP-independent structured command (FR-071).
`enterprise_id`, `recommendation_id`, `capability_id`, `action_type` (`create_po`),
`idempotency_key text unique not null`, `params jsonb`
(`{item_id, supplier_id, quantity, unit_price, currency, needed_by}`),
`lorm_level_at_decision` (`L3`|`L4`|`L5`),
`enforcement_decision jsonb` (`{decision, effective_level, reason, policy_id, policy_version}`),
`status` (see state machine), `dispatched_at`, `adapter_type`, `external_ref`,
`result jsonb`, `last_error text`.

**State machine (`status`)**:

```
prepared ──(L3)──────────────► recommendation_only        (terminal; no execution)
prepared ──(L4, approve)─────► authorized ──► dispatching ──► sent
prepared ──(L4, reject)──────► rejected                    (terminal; reason stored)
prepared ──(L5, in-bounds)───► authorized ──► dispatching ──► sent
prepared ──(L5, out-of-bounds)► demoted_to_l4  ──► (enters L4 approval queue)
sent ──(adapter ack)─────────► executed
sent ──(adapter error)───────► failed
sent ──(crash/restart)───────► sent_unknown ──(reconcile: get_status)──► executed | failed
authorized/prepared (undispatched, durable) ──(restart)──► re-enqueued, unchanged
```

`executed` and `failed` are followed by the verification lifecycle (§8).

---

## 7. LORM — capabilities, levels, policies (`lorm/`, `policies/`)

### `capability`
`enterprise_id`, `key` (one of the 8: `proc.inventory.observe`, `proc.demand.observe`,
`proc.risk.diagnose`, `proc.order.recommend`, `proc.po.create`, `proc.replenish.routine`,
`proc.supplier.add`, `proc.payment.release`),
`level` (`L0`..`L5`), `l5_allowed bool` (false for `proc.supplier.add` and
`proc.payment.release` — FR-049; enforced by CHECK + seed),
`verification_tolerances jsonb` — **v1 defaults `{price_pct: 10, qty_short_pct: 10,
late_days: 3}`**; supplier mismatch is always material; overridable per capability or per
policy (product/demo defaults, not LORM-normative — FR-055a),
`uncertainty_threshold numeric` (default 0.3).

**Seed levels (fixed in `lorm/seed.py`, tasks.md T021)**: `proc.inventory.observe`=L0,
`proc.demand.observe`=L1, `proc.risk.diagnose`=L2, `proc.order.recommend`=L3,
`proc.po.create`=L3, `proc.replenish.routine`=L4, `proc.supplier.add`=L4 (`l5_allowed=false`),
`proc.payment.release`=L3 (`l5_allowed=false`). Story tests set/promote the level they need
explicitly.

`level` is only ever **raised** by an explicit human action through `capability_service`
(one step; the minimal version lives in the Foundational phase); only ever **lowered** by
`demotion_event` (one step).

### `capability_level_event` (append-only) — unified history (FR-050/FR-058)
`capability_id`, `direction` (`promotion`|`demotion`), `from_level`, `to_level`,
`reason text`, `trigger` (`human`|`incident`|`rollback`|`verification_failure`|
`observability_loss`|`uncertainty`|`policy_expiry`|`policy_revocation`),
`actor_id` (null for automatic), `at`.

### `promotion_request`
`capability_id`, `proposed_to_level`, `rationale text`, `evidence jsonb`
(trust record / L4 track record), `origin` (`ai`|`human`),
`status` (`pending`|`approved`|`rejected`), `decided_by`, `decided_at`.
Approval applies the change (one level). For `L4→L5`, `decided_by` MUST differ from the
capability's active policy author (SPEC §6.2). AI `origin` never auto-applies (I-8, FR-052).

### `policy`  (canonical L5 policy; serializes to a doc validating against
`vendor/lorm/lorm-policy.schema.json`)
`enterprise_id`, `capability_id`, `version int`, `title`,
`bounds jsonb` (`{skus[], categories[], suppliers[], max_order_amount, currency,
period_aggregate:{window, max_amount}, extra{}}`),
`conditions jsonb` (list of `{text, check?}`),
`verification jsonb` (`{expect, window, tolerances_override?}`),
`author_id`, `approved_by_id` (null until approved), `approved_at`,
`expires_at`, `tested text`,
`status` (`draft`|`pending_approval`|`active`|`rejected`|`revoked`|`expired`),
`revoked_by`, `revoked_at`, `schema_valid bool`, `schema_errors jsonb`.

**Constraints**: `author_id <> approved_by_id` (FR-038, DB CHECK + service check);
`capability.l5_allowed = true` required to create (FR-049);
`validate_policy.py` must pass (`schema_valid = true`) before `pending_approval` (FR-036/FR-044).

**State machine**:

```
draft ──► pending_approval ──(approver ≠ author, Approve)──► active
                            └──(Reject)───────────────────► rejected
active ──(expires_at reached)──► expired        (capability L5→L4, automatic)
active ──(revoke)─────────────► revoked         (capability L5→L4, automatic)
active ──(edit bounds)────────► new version in draft (re-approval required)
```

### `policy_approval` (append-only)
`policy_id`, `policy_version`, `decision` (`approve`|`reject`), `actor_id`, `reason text`, `at`.

### `policy_draft_suggestion`
`capability_id`, `suggested jsonb` (AI-proposed bounds/conditions), `llm_provider`,
`llm_model`, `created_at`, `consumed_by_policy_id` (nullable). Advisory only (FR-037).

---

## 8. Execution & verification (`execution/`, `verification/`)

### `execution_attempt` (append-only)
`procurement_action_id`, `adapter_type`, `idempotency_key`, `request jsonb`,
`response jsonb`, `outcome` (`ack`|`error`|`status_query`), `at`.

### `verification_result`
`procurement_action_id`, `capability_id`,
`status` (`pending`|`verified`|`failed`|`unverifiable`),
`window_expires_at`,
`measurements jsonb` (`{price:{expected,actual,deviation_pct,material},
qty:{expected,received,short_pct,material}, lead:{expected_at,received_at,late_days,material},
supplier:{expected_id,actual_id,material}}`),
`overall_material bool`, `explained_accepted_by` (nullable FK user), `reason text`,
`evaluated_at`.

**State**: `pending` → `verified` | `failed` | `unverifiable`.
`failed` & `overall_material` & not `explained_accepted_by` ⇒ triggers `demotion_event`
(trigger = `verification_failure`) (FR-056, SPEC §6.3/§10.1).
Repeated `unverifiable` for a capability ⇒ surfaced to approver, blocks L5 eligibility
(SPEC §10.1).

### `demotion_event` — represented by `capability_level_event` rows with `direction=demotion`
(kept as one history table; no separate table needed).

### `execution_attempt` append-only note
`execution_attempt` (and every other table marked *append-only* in this document —
`audit_record`, `capability_level_event`, `mapping_change_event`, `observation_signal`,
`policy_approval`) is protected at the database level by a reusable `forbid_mutation()` PL/pgSQL
trigger that raises on UPDATE/DELETE (tasks.md T027, FR-063, Principle X). Application-level
`AppendOnly` assertions remain as defense in depth.

### Spec-entity mapping (no dedicated tables)
- **"Approval (L4)"** (spec Key Entities) is not a table. An L4 approve/reject is the
  `procurement_action` state transition (`prepared → authorized` or `prepared → rejected`,
  with the rejection reason on the action) **plus** append-only `audit_record` events
  `l4_approved` / `l4_rejected` carrying `authorizer` (the user) and timestamp. A dedicated
  `approval` table is added only if implementation requires it.
- **"Execution / Order"** (spec Key Entities) maps to `procurement_action` + its
  `execution_attempt` rows (and, for the simulated adapter, `simulated_order`). There is no
  separate `order` table.

---

## 9. Audit (`audit/`)

### `audit_record` (append-only; INSERT/SELECT only for app role)
Superset of LORM §10.3 + spec FR-060/FR-061:
`enterprise_id`, `at`,
`capability_id`, `capability_key`, `lorm_level` (level after any degradation),
`event_type` (`recommendation`|`l4_prepared`|`l4_approved`|`l4_rejected`|
`l5_authorized`|`dispatched`|`execution_result`|`verification`|`promotion`|`demotion`|
`recovery`|`ai_unavailable`|`policy_approved`|`policy_revoked`|`mapping_confirmed`),
`decision jsonb` (the proposed/actioned decision),
`evidence_ref jsonb` (`diagnosis_ref` → risk_finding + signal links),
`authorizer jsonb` (`{kind:"human", user_id}` for L4 | `{kind:"policy", policy_id, version}` for L5),
`action text`, `params jsonb`,
`procurement_action_id` (nullable), `outcome jsonb`,
`verified` (`verified`|`failed`|`unverifiable`|`pending`|`n/a`),
`correlation_id uuid`.
Retention per enterprise config. Not derived from LLM context (FR-063).

---

## 10. Background jobs (`jobs/`)

### `job` (durable queue — research.md §3)
`enterprise_id` (nullable for global), `kind` (`observe_source`|`recompute_aggregates`|
`detect_risks`|`generate_explanation`|`generate_recommendation`|`prepare_action`|
`dispatch_action`|`reconcile_executions`|`run_verification`|`policy_expiry_scan`|
`suggest_mapping`), `payload jsonb`, `run_at`, `status` (`pending`|`running`|`done`|`failed`),
`attempts int`, `max_attempts int`, `locked_by text`, `locked_at`, `last_error text`.
Enqueued in the same transaction as the domain write that schedules it. Handlers are
idempotent. Recurring jobs re-enqueue their next run on completion.

---

## 11. Relationship overview

```
enterprise 1─* user *─* role *─* permission
enterprise 1─* data_source 1─* source_field
                     1─* field_mapping 1─* mapping_change_event
                     1─? secret
data_source 1─* observation_signal
item 1─* stock_level / price / lead_time / consumption / item_supplier / quality_record
supplier 1─* item_supplier / price / lead_time / quality_record
item 1─1 sku_aggregate(*by warehouse)
risk_finding *─* observation_signal   (risk_signal_link)
risk_finding 1─1 explanation(subject=risk_finding)
risk_finding 1─* recommendation 1─1 explanation(subject=recommendation)
recommendation 1─* procurement_action 1─* execution_attempt
procurement_action 1─* verification_result
capability 1─* capability_level_event
capability 1─* promotion_request
capability 1─* policy 1─* policy_approval
capability 1─* policy_draft_suggestion
* many operations ─* audit_record   (via correlation_id + explicit FKs where 1:1)
```

---

## 12. Validation rules (from requirements)

| Rule | Source | Enforced where |
|---|---|---|
| Only `confirmed` field mappings used by sync | FR-004/FR-005 | `integration/` sync query filter |
| `policy.author_id <> approved_by_id` | FR-038 | DB CHECK + `policies/` service |
| L5 policy blocked for `l5_allowed=false` capabilities | FR-049 | CHECK + seed + `policies/` service |
| Policy must pass `validate_policy.py` before `pending_approval` | FR-036/FR-044 | `policies/` service |
| Editing bounds creates a new `version` needing re-approval | FR-044 | `policies/` service |
| `capability.level` raised only via approved `promotion_request`, one step | FR-051/I-8 | `lorm/` service; no other writer |
| `capability.level` lowered automatically one step on §6.3 triggers | FR-056/FR-057 | `lorm/demotion.py` |
| `execution.dispatch()` requires a valid `EnforcementDecision` | FR-069/§7 | function signature guard |
| Action state persisted before dispatch; idempotent send; reconcile on restart | FR-032a | `execution/` + `reconcile_executions` job |
| Every L4/L5 `audit_record` has evidence_ref + authorizer + verified | FR-060/FR-061, I-6 | `audit/` writer asserts required fields |
| Append-only tables reject UPDATE/DELETE | FR-063, Principle X | `forbid_mutation()` DB trigger (tasks.md T027) + `AppendOnly` app assertions (defense in depth) |
| Verification "material" thresholds | FR-055a | `capability.verification_tolerances` defaults `{price_pct:10, qty_short_pct:10, late_days:3}`, supplier mismatch always material; per-capability / per-policy override in `verification/tolerances.py` |
| Low `confidence` / high `uncertainty` ⇒ escalate, never autonomous | FR-025, I-3 | `LormEnforcementService.evaluate()` |
| AI outputs validated against Pydantic schema; invalid ⇒ fail-safe | FR-016a, FR-071 | `ai/` provider wrapper |
| Secrets never in API responses / prompts / plaintext columns | FR-068, §15 | `secret` table shape + serializer allowlist |
