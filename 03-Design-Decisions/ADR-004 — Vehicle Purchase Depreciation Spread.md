# ADR-004 — Vehicle Purchase Depreciation Spread

#decision #module/cost-ledger #module/vehicle-maintenance #status/active

## Status

Accepted

## Context

When a logistics company buys a vehicle for 1,000,000 on May 5th, recording it as a single cost event creates a massive spike — May shows 1M, June shows 3K (just maintenance). This makes monthly trend analysis useless and delivery cost allocation absurd (deliveries on May 5th would bear the entire purchase cost).

## Decision

**A vehicle purchase creates two things:**

1. A **[[Cost Event]]** for the full purchase amount (for accounting/finance — the money did leave the account on that date)
2. A **[[Cost Rule]]** for daily depreciation that spreads the cost over the vehicle's expected useful life

```
Purchase: 1,000,000 on 2025-05-05
Expected life: 5 years (1,826 days)

Cost Event:
  entity=vehicle-001, cost_type="purchase", amount=1000000, occurred_on=2025-05-05

Depreciation Rule:
  entity=vehicle-001, cost_type="depreciation"
  amount = 1000000 / 1826 ≈ 547.70/day
  frequency = daily
  effective_from = 2025-05-05
  effective_until = 2030-05-04
```

After the expected life date, the depreciation rule naturally expires. The vehicle's fixed cost contribution drops to zero (though maintenance/insurance rules may still be active).

## How This Affects Different Views

**Finance monthly report ([[ADR-005 — Finance vs Operations Cost Views]]):**

- Shows the 1M purchase event in May (actual cash flow)
- Also shows depreciation as a separate line item
- Finance team chooses which view to use for which report

**Per-Work-Order cost allocation ([[Integration — Order × Cost Ledger]]):**

- Uses the depreciation rule (547.70/day) for fixed cost allocation
- Never uses the purchase event directly
- Result: delivery costs are stable and comparable across months

**Vehicle TCO dashboard:**

- Can show either view or both
- Depreciation gives the "true cost of operating this vehicle per month" view

## Expected Life and Residual Value

For now, we use straight-line depreciation with zero residual value. Future enhancements:

- Allow residual value: `daily_depreciation = (purchase_price - residual_value) / expected_life_days`
- Support declining balance method via a different rule type

## Disposal

Per [[ADR-018 — Negative Amounts and Disposal]]:

```
Vehicle V1 sold for 200,000 with 400,000 remaining book value:

1. Close depreciation rule:
     UPDATE CostRule SET effective_until = sale_date
     WHERE entity_id = V1 AND cost_type = depreciation

2. Record sale proceeds (negative cost = money in):
     INSERT CostEvent(entity_id=V1, cost_type=disposal_sale,
                      amount=-200000, classification=one_off,
                      occurred_on=sale_date)

3. (Optional) Record write-off for explicit accounting:
     INSERT CostEvent(entity_id=V1, cost_type=write_off,
                      amount=400000, classification=one_off,
                      occurred_on=sale_date)
```

The unrecovered book value (400,000) is recorded only if the operator wants explicit P&L visibility on the disposal loss. Without it, depreciation simply stops and the loss is implicit in the absence of further depreciation accruals.

Insurance payouts on a vehicle (claim received) follow the same pattern: `CostEvent` with negative `amount`, `classification=one_off`.

## User Input Required

When recording a vehicle purchase, the user must provide:

- Purchase price
- Expected useful life (years or days)
- (Future) Residual value estimate

The system auto-creates both the cost event and the depreciation rule.

## Consequences

- **Good:** Monthly trends are meaningful — no purchase spikes
- **Good:** Delivery cost allocation is stable
- **Good:** Vehicle fixed cost naturally drops to zero after expected life
- **Good:** Finance still has the actual purchase event for accounting
- **Tradeoff:** Two records per purchase (event + rule) — minor complexity
- **Tradeoff:** Expected life is an estimate — may need adjustment later (close old depreciation rule, create new one with remaining value)