# ADR-017 — Append-Only CostEvents

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

[[ADR-001 — Close and Replace, Never Mutate]] applies to [[Cost Rule]]s. The [[Cost Event]] doc previously offered two correction paths:

> Mutate directly (events are point-in-time facts, less risk than mutating rules)
> Or for full audit trail: soft-delete the wrong event + create corrected one

Meanwhile, [[Integration — Fuel Log × Cost Ledger]] uses a third pattern: append a reversing CostEvent.

Three policies for the same primitive is a recipe for drift. Audit history depends on reproducibility — running totals must reconcile by the same rule everywhere.

## Decision

**[[Cost Event]] is append-only. Corrections are reversing events, not mutations. Aligns with [[ADR-001]] philosophy across all ledger writes.**

### Correction pattern

```
Original event: amount = 2,000 (fuel log error)

Correction:
  1. INSERT CostEvent(amount = -2000, source_type = reversal,
                      source_ref  = original_event_id,
                      cost_classification = same as original)
  2. INSERT CostEvent(amount =  1800, source_type = correction,
                      source_ref  = original_event_id,
                      cost_classification = same as original)

Sum across all three events: 2000 - 2000 + 1800 = 1800 ✓
```

### Source tracking

Every CostEvent has:

| Field | Purpose |
|---|---|
| `source_type` | `manual` / `fuel_log` / `maintenance` / `repair` / `reversal` / `correction` / `mobile_entry` |
| `source_ref` | ID linking to the originating record (FuelLog id, original CostEvent id for reversals, etc.) |

A reversing event always has `source_type = reversal` and `source_ref = original_event_id`. A correction (the new "what should have been recorded") has `source_type = correction` and `source_ref = original_event_id`.

### Soft-delete on the source record

If a [[Fuel Log]] is deleted, the FuelLog row is soft-deleted (status=deleted) AND a reversing CostEvent is appended. The CostEvent itself is never hard-deleted.

### No mutation path

The "mutate directly" path in [[Cost Event]] is removed. There is no UPDATE on `amount`, `occurred_on`, `entity_id`, `entity_type`, `cost_type`, `cost_classification`, or `work_order_id`. Audit-only fields (`note` text, soft-deletion flag) may be edited.

### Sum-friendly running totals

```
net_cost(filter) = Σ CostEvent.amount WHERE filter
```

No need to filter "active" rows. Reversals net to zero; corrections add the new amount on top. Reports can sum naively.

## Consequences

- **Good:** Single rule across all CostEvents — no special-casing
- **Good:** Running totals always reconcile by simple sum (no need to filter)
- **Good:** Full audit history — reversal chain is explicit
- **Good:** Matches [[ADR-001]] philosophy uniformly across Rule and Event
- **Tradeoff:** More events stored (negligible at typical fleet scale)
- **Tradeoff:** UI must compute "current effective amount" by looking at latest non-reversed correction, not by filtering rows

## Related

- [[Cost Event]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-018 — Negative Amounts and Disposal]]
- [[Integration — Fuel Log × Cost Ledger]]
