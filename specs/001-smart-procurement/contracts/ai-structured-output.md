# Contract — AI Structured Output

Every LLM interaction goes through `LLMProvider.generate_structured(output_schema, system,
prompt, context) -> BaseModel`. The raw text is parsed and **validated against the Pydantic
schema below**; a `ValidationError`, timeout, or provider error is handled exactly like AI
unavailability (retry/backoff → degraded "human-required" → block dependent L5 → audit;
FR-016a). The LLM never returns an action to execute — only analysis/among/explanation.

Common envelope fields on every schema:
`model_meta: { provider: str, model: str, generated_at: datetime, token_usage?: {...} }`.

---

## 1. `MappingSuggestion`  (source onboarding, FR-003/FR-004)

```
MappingSuggestionSet:
  suggestions: list[MappingSuggestion]        # one per source field it can classify
MappingSuggestion:
  source_field_path: str                      # must exist in the introspected source_field set
  canonical_entity: enum(item, warehouse, stock_level, supplier, item_supplier,
                         price, lead_time, purchase_order, consumption,
                         production_demand, quality_record)
  canonical_attribute: str                    # e.g. "sku", "quantity", "unit_price"
  transform: object | null                    # optional normalization hint
  confidence: float                           # 0..1
  reasoning: str                              # short, human-readable
```
**Backend rules**: unknown `source_field_path` → suggestion dropped; nothing is applied —
each becomes a `field_mapping(status=suggested)` awaiting human `confirm` (FR-004).

---

## 2. `RiskExplanation`  (L2, FR-018/FR-019/FR-070, invariants I-2/I-3)

```
RiskExplanation:
  what: str
  why: str                                    # why this is a problem
  data_used: list[EvidenceRef]                # MUST be non-empty (I-1)
  factors: list[{ name: str, effect: str, weight: enum(low,med,high) }]
  confidence: float                           # 0..1  (I-3: mandatory)
EvidenceRef:
  kind: enum(observation_signal, sku_aggregate, domain_row)
  ref: str                                    # id / natural key resolvable by backend
```
**Backend rules**: `data_used` referencing ids not linked to the `risk_finding` are rejected
(anti-hallucination); empty `data_used` ⇒ treated as invalid output ⇒ `ai_status=unavailable`.

---

## 3. `ProcurementRecommendation`  (L3, FR-021–FR-024, FR-070)

```
ProcurementRecommendation:
  item_ref: str                               # must equal the risk's item
  proposed_supplier_ref: str                  # must be an approved supplier for the item
  quantity: number                            # > 0
  needed_by: date
  expected_unit_price: number
  currency: str                               # = enterprise base_currency
  expected_outcome: str
  reasons: list[str]                          # non-empty
  risks: list[str]
  alternatives: list[Alternative]             # >= 1 when >1 eligible supplier exists
  do_nothing_consequence: str | null
  confidence: float                           # 0..1
Alternative:
  supplier_ref: str
  quantity: number
  unit_price: number
  lead_time_days: number
  why_not: str
```
**Backend rules**: `proposed_supplier_ref` not in the item's approved-supplier list → invalid
(the model may not widen supplier scope — mirrors the LORM `proc.supplier.add` self-permission
prohibition). `quantity <= 0`, wrong currency, or missing `reasons` → invalid. `confidence`
below the capability's escalation threshold ⇒ recommendation is presented but **not** eligible
for autonomous (L5) execution (FR-025).

---

## 4. `PolicyDraft`  (L5 authoring assistance, FR-037)

```
PolicyDraft:
  capability_key: str
  bounds: { skus?: list[str], categories?: list[str], suppliers?: list[str],
            max_order_amount?: number, currency?: str,
            period_aggregate?: { window: str, max_amount: number }, extra?: object }
  conditions: list[{ text: str, check?: str }]
  verification: { expect: str, window: str, tolerances_override?: object }
  rationale: str
```
**Backend rules**: purely advisory — stored as `policy_draft_suggestion`. It is **not** a
policy, cannot be submitted or approved by AI, and still requires a human author + a distinct
human approver and a passing `validate_policy.py` run before it can reach `pending_approval`
(FR-037/FR-038, I-8).

---

## 5. Provider abstraction contract

```
class LLMProvider(Protocol):
    async def generate_structured(
        self, output_schema: type[BaseModelT], *,
        system: str, prompt: str, context: dict, timeout_s: float,
    ) -> BaseModelT: ...
```
- Implementations: `AnthropicProvider` (real; latest Claude model), `DeterministicMockProvider`
  (fixtures keyed by task + input hash — used by the default test suite).
- The wrapper enforces: retry with exponential backoff (configurable attempts), total
  timeout, schema validation, and emission of an `ai_unavailable` audit record on final
  failure. Secrets/credentials are never placed in `system`, `prompt`, or `context`
  (FR-068).
