# Contract — LORM Enforcement

`LormEnforcementService` is the in-process adaptation layer (research.md §6). It preserves the
reference implementation's three outcomes (`allow` / `ask` / `deny`) and invariants
I-1…I-8. The canonical L5 policy document validates against the **vendored, unmodified**
`vendor/lorm/lorm-policy.schema.json`.

## Enforcement call

```python
class LormEnforcementService(Protocol):
    async def evaluate(self, req: EnforcementRequest) -> EnforcementDecision: ...
```

```
EnforcementRequest:
  enterprise_id: uuid
  capability_key: str
  proposed_action: { action_type: "create_po",
                     params: {item_id, supplier_id, quantity, unit_price, currency, needed_by} }
  evidence_ref: { risk_finding_id: uuid, signal_ids: list[uuid] }   # L2 diagnosis (I-1)
  uncertainty: float                                                # from recommendation.confidence → 1-confidence, or explanation
  requested_by: { kind: "human"|"system", user_id?: uuid }

EnforcementDecision:
  decision: enum(allow, ask, deny)
  effective_level: enum(L0,L1,L2,L3,L4,L5)      # level the action would run at (after degradation)
  reason: str                                   # human-readable, stored in audit
  policy_id: uuid | null                        # set when decision derived from an L5 policy
  policy_version: int | null
  degraded_from: enum(...) | null               # set when an L5 request was degraded to ask/deny
  decision_token: str                           # opaque HMAC over (procurement_action_id, capability, params hash, decision)
```

`execution.dispatch()` accepts only a `decision == allow` whose `decision_token` matches the
exact `procurement_action` (FR-069). `ask` routes the action into the L4 approvals queue;
`deny` marks it `recommendation_only`.

## Decision rules (mirrors SPEC.md)

1. **I-1 gate** — `evidence_ref` missing, or its `risk_finding` has `ai_status=unavailable`,
   or a linked signal's source is `unavailable`/`stale` beyond tolerance → `deny`
   (`reason="no current L2 diagnosis grounded in L1 observations"`).
2. **Capability level cap** —
   - `capability.level ∈ {L0,L1,L2,L3}` → `deny`, `effective_level = L3` (recommend only).
   - `capability.level == L4` → `ask`, `effective_level = L4`.
   - `capability.level == L5` → continue to policy evaluation.
3. **I-3 / uncertainty** — `uncertainty > capability.uncertainty_threshold` (or the active
   policy's threshold) → degrade one step: L5→`ask` (L4). Record `degraded_from`.
4. **L5 policy evaluation** — load the **single `active`** `policy` for the capability:
   - none active, or `expires_at <= now()`, or `status != active` → `ask` (L4),
     `degraded_from="policy_missing_or_expired"`; also enqueue `policy_expiry_scan` follow-up.
   - `l5_allowed == false` for the capability → `deny` (config error; should never have a
     policy — `proc.supplier.add`, `proc.payment.release`).
   - **bounds check** against `proposed_action.params`:
     - `item` sku ∈ `bounds.skus` OR `item.category` ∈ `bounds.categories` (if either set).
     - `supplier` ∈ `bounds.suppliers` (if set) AND supplier is approved for the item.
     - `quantity * unit_price <= bounds.max_order_amount`.
     - period aggregate: `sum(executed L5 orders for this capability within
       bounds.period_aggregate.window) + this order <= bounds.period_aggregate.max_amount`
       (computed from `audit_record` / `procurement_action`, per SPEC "rate from audit log").
   - any bound fails → `ask` (L4), `degraded_from="out_of_bounds"` (FR-042 — never silent).
   - **conditions** — for each `conditions[i]` with a `check`, run it (short timeout); a
     non-zero exit or timeout → degrade to `ask` (L4), `degraded_from="condition_failed"`.
   - all pass → `allow`, `effective_level = L5`, `policy_id`/`policy_version` set.

## Automatic demotion (runtime; SPEC §6.3, FR-056/FR-057)

`lorm/demotion.py` decrements `capability.level` by **exactly one** and writes a
`capability_level_event(direction=demotion, trigger=...)` — **without** human approval — on:

| trigger | raised by |
|---|---|
| `incident` / `rollback` | `execution/` (adapter-reported incident or an invoked reversal) |
| `verification_failure` | `verification/` (`failed`, `overall_material`, not `explained_accepted_by`) |
| `observability_loss` | `analysis/` / `integration/` (`observability_gap` opened for the capability's inputs, incl. `ai_unavailable`) |
| `uncertainty` | `analysis/` (sustained uncertainty over threshold) |
| `policy_expiry` / `policy_revocation` | `policy_expiry_scan` job / `/policies/{id}/revoke` |

Editing the vendored policy document's `demotions:` array remains human-only (I-8). Promotion
is always a human-approved `promotion_request`, one level, and L4→L5 needs an approver other
than the policy author (SPEC §6.2).

## Audit obligation (I-6, §10.3)

Every `evaluate()` that yields `ask`/`allow`, every dispatch, execution result, verification,
and every level change writes an `audit_record` with: `capability`, `lorm_level` (effective),
`authorizer` (`{human,user_id}` for L4 approval | `{policy_id,version}` for L5), `action`,
`params`, `evidence_ref` (→ `diagnosis_ref`), `outcome`, `verified`, `correlation_id`
(data-model §9).

## Reuse boundary (for `docs/lorm-integration.md`)

| Reused unmodified | Adapted | Built here |
|---|---|---|
| `lorm-policy.schema.json`, `validate_policy.py`, `SPEC.md` semantics, procurement example seeds | enforcement outcomes as an in-process service instead of Claude-Code `hooks/` | `LormEnforcementService`, `demotion.py`, policy CRUD/approval, capability level store, bounds/period-aggregate evaluation over our audit log |
