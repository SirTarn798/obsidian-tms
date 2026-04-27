# Monthly Operational Cost

#module/cost-ledger #status/active

> The overhead bucket. Costs of running the business that are **not** allocated to specific Work Orders — office staff salary, rent, utilities, fleet-wide subscriptions, facility costs.

## Context

Per-Work-Order cost ([[Integration — Order × Cost Ledger]]) answers "what did this trip cost?" — vehicle fixed share, variable cost (fuel/maintenance/tire), driver per-trip + per-km + salary share, per-WO incidentals (tolls, parking).

Monthly Operational Cost answers a narrower, complementary question: **"what overhead do we bear each month that isn't tied to any single trip?"**

These are the costs you'd pay even if no truck moved this month — controllers, planners, accountants, the office lease, electricity, software subscriptions. Real money, real obligations, but no honest allocation key to per-trip economics.

The two views are deliberately disjoint. Together they cover the operating side of the business; per-trip cost + overhead = total operating cost. Cash-flow events (vehicle purchase, disposal sale) and per-trip incidentals are **not** in either view — see [[ADR-005 — Finance vs Operations Cost Views]] and [[ADR-018 — Negative Amounts and Disposal]].

## What's Included

| Bucket | Source |
|---|---|
| Office staff salaries | [[Cost Rule]] `entity_type=user, cost_classification=fixed_time` (controller, planner, accountant, owner) |
| Office rent | [[Cost Rule]] `entity_type=facility, cost_classification=fixed_time` |
| Utilities, internet, software | [[Cost Rule]] `entity_type=facility, cost_classification=fixed_time` |
| Fleet-wide subscriptions (GPS, dispatch software) | [[Cost Rule]] `entity_type=facility, cost_classification=fixed_time` |
| Office bonuses, one-off facility costs | [[Cost Event]] `entity_type ∈ (user, facility), cost_classification=one_off` |

## What's NOT Included

| Cost | Where it lives instead |
|---|---|
| Vehicle depreciation, insurance, registration | Per-WO `fixed_cost_share` |
| Fuel, maintenance, tire | Per-WO `variable_cost` via [[Cost Per Km]] |
| Driver salary | Per-WO via duration share ([[ADR-019 — Driver Salary Allocation by Duration Share]]) |
| Driver per-trip / per-km | Per-WO `driver_cost` via [[Driver Rate]] |
| Tolls, parking, port fees | Per-WO `events_cost` ([[ADR-015 — Per-WO Cost Events]]) |
| Vehicle purchase (cash event) | Cash flow only — not in any operating view |
| Vehicle disposal sale, insurance payout | Cash flow only — not in any operating view |
| Driver idle hours | Fleet utilization metric (separate dashboard line) |

## Computation

```
monthly_operational_cost(month) =
    Σ active CostRule (entity_type ∈ {user, facility}, cost_classification=fixed_time)
        prorated to days in month
  + Σ CostEvent (entity_type ∈ {user, facility}, cost_classification=one_off)
        WHERE occurred_on IN month
```

Filter is by `entity_type` AND `cost_classification`. This avoids accidentally including per-WO incidentals (`entity_type=vehicle, classification=one_off, work_order_id=...`) which belong on a specific WO.

Daily proration follows [[ADR-003 — Month-by-Month Proration]] (uses each month's actual `days_in_month` for accurate Feb-vs-Dec rates).

## Reports

```
GET /reports/monthly_operational_cost?month=2026-04
→ {
    month: "2026-04",
    office_salaries:     ...,    -- entity_type=user
    facility_fixed:      ...,    -- entity_type=facility, fixed_time rules
    facility_one_off:    ...,    -- entity_type=facility, one_off events
    total: ...
  }
```

A drill-down per-entity view is available for finance to audit individual obligations.

## Why office salary lives here, not in WO cost

A Work Order is a single trip. Charging a portion of the controller's salary to a specific trip would require an arbitrary allocation key (per-WO? per-mile? per-revenue?) — none of which reflect what controllers actually do. Their work spans planning, customer relations, fleet management. It is a real cost, but a **business-level** cost.

Per [[ADR-013 — Driver as Cost Entity, Not Employee]], the per-WO cost model is intentionally narrow (vehicle + driver, with `work_order_id`-tagged incidentals). Everything else is captured one level up, in this view.

## Related

- [[Cost Ledger — Overview]]
- [[Cost Rule]]
- [[Cost Event]]
- [[Integration — User × Cost Ledger]]
- [[Integration — Order × Cost Ledger]]
- [[ADR-003 — Month-by-Month Proration]]
- [[ADR-005 — Finance vs Operations Cost Views]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-014 — Cost Classification]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
