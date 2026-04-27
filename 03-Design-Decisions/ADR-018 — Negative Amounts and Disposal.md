# ADR-018 — Negative Amounts and Disposal

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

The original Cost Ledger design assumed `amount > 0` on all CostEvents — costs only flow out. But the system must handle:

- **Vehicle disposal sale.** Truck sold for cash; that cash partially offsets prior depreciation.
- **Insurance payouts.** Claim paid out reduces net vehicle cost.
- **Refunds.** A returned part, an overcharged invoice corrected.
- **Reversals.** [[ADR-017 — Append-Only CostEvents]] requires negative-amount reversing events.

Modeling these as a separate "IncomeEvent" or "Refund" table fragments the ledger and breaks the centralization goal. A separate `direction` enum complicates aggregation.

## Decision

**[[Cost Event]] `amount` may be negative. The ledger is net-cost: positive = money out, negative = money in. `SUM(amount)` over any filter produces the net cost for that filter.**

No separate income store. No `direction` field. Sign of `amount` carries the meaning.

### Use cases

| Scenario | Event |
|---|---|
| Vehicle disposal sale, 200,000 received | `CostEvent(entity_type=vehicle, cost_type=disposal_sale, amount=-200000, classification=one_off)` |
| Insurance payout, 50,000 received | `CostEvent(entity_type=vehicle, cost_type=insurance_payout, amount=-50000, classification=one_off)` |
| Reversal of erroneous 2,000 fuel entry | `CostEvent(entity_type=vehicle, cost_type=fuel, amount=-2000, classification=variable_usage, source_type=reversal, source_ref=original_id)` |
| Refund of overcharged repair, 5,000 | `CostEvent(entity_type=vehicle, cost_type=repair_refund, amount=-5000, classification=one_off)` |
| Toll refund attributed to WO-77, 50 | `CostEvent(entity_type=vehicle, cost_type=toll_refund, amount=-50, classification=one_off, work_order_id=WO-77)` |

### Disposal lifecycle

```
Vehicle V1 sold for 200,000 with 400,000 remaining book value:

1. Close depreciation rule:
     UPDATE CostRule SET effective_until = sale_date
     WHERE entity_id = V1 AND cost_type = depreciation

2. Record sale proceeds:
     INSERT CostEvent(entity_id=V1, cost_type=disposal_sale,
                      amount=-200000, classification=one_off,
                      occurred_on=sale_date)

3. (Optional) Record write-off for explicit accounting:
     INSERT CostEvent(entity_id=V1, cost_type=write_off,
                      amount=400000, classification=one_off,
                      occurred_on=sale_date,
                      note="Unrecovered book value at disposal")
```

The unrecovered book value (400,000) is **optionally** recorded as a `write_off` event. If not recorded, the depreciation simply stops — no implicit loss recognition. Operators choose based on accounting needs. Both flows produce the same operational outcome (vehicle no longer in service); the difference is whether the loss is explicit in P&L.

### Reversal hygiene

Reversing events follow [[ADR-017 — Append-Only CostEvents]]:

- Same `cost_classification` as the original
- Same `entity_type` and `entity_id` as the original
- Opposite sign on `amount`
- `source_type = reversal`
- `source_ref = original_event_id`

Reversals net to zero in any sum that includes both events.

### View interaction

| View | Negative amounts |
|---|---|
| [[Cost Per Km]] | Sums variable_usage events including reversals. A 2,000 fuel + (-2,000) reversal contributes net 0 to the rate. |
| [[Monthly Operational Cost]] | Sums overhead-classification rules + events. Refunds and payouts net out. |
| Per-WO `events_cost` | Negative values allowed. A toll refund attributable to a WO reduces that WO's events_cost. |
| Cash flow export | Signed amounts preserved; positive=out, negative=in. |

## Consequences

- **Good:** Single ledger, one query interface — no income/refund/cost split
- **Good:** Disposal accounting is explicit and computable
- **Good:** Reversals are unified with disposals/refunds — same mechanism
- **Good:** Insurance payouts reduce net vehicle cost without special-casing
- **Tradeoff:** UI must distinguish "out" from "in" visually (e.g. sign or color) without separate tables
- **Tradeoff:** Some external accounting export formats expect signed-positive convention — translation layer at export, not at storage

## Related

- [[Cost Event]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-004 — Vehicle Purchase Depreciation Spread]]
- [[ADR-017 — Append-Only CostEvents]]
