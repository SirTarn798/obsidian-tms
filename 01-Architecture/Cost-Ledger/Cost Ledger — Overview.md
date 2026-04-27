# Cost Ledger

#module/cost-ledger #status/active

> Event-sourced cost tracking system that records what happened and when, then computes any cost view on demand. Single source of truth for managerial cost — all transport, driver, and overhead costs flow through one set of primitives.

## Context

The platform needs to track costs across multiple entity types ([[Vehicle]], [[Driver]], User, facility, future: equipment, etc.) with full historical accuracy. Costs are messy in practice — drivers join mid-month, vehicles are purchased on arbitrary dates, salaries change, maintenance is lumpy, customers demand per-trip incidentals. The previous system scattered cost across `t_route_cost`, `t_sub_cost`, `t_tms_cost`, `t_tms_cost_drop`, `t_calculate_costs`, `t_average_costs`, `t_employee_salary_history` and several maintenance/fuel domain tables. The redesign collapses all of this into two primitives plus computed views.

## How It Works

The system has three layers:

### Layer 1 — Ledger (Source of Truth)

Two primitives store everything:

- **[[Cost Rule]]** — An ongoing cost obligation with a time range. "From date X to date Y, this entity costs Z per period." Rules are never mutated — they are closed and replaced. See [[ADR-001 — Close and Replace, Never Mutate]].
    
- **[[Cost Event]]** — A point-in-time cost. Purchases, repairs, fuel refills, tolls, bonuses, disposals. Append-only with reversing-event corrections. May carry a `work_order_id` for per-trip incidentals. May be negative (refunds, payouts, disposals). See [[ADR-017 — Append-Only CostEvents]] and [[ADR-018 — Negative Amounts and Disposal]].

Both primitives carry a `cost_classification` field (`fixed_time` / `variable_usage` / `one_off`) that drives view selection — see [[ADR-014 — Cost Classification]].

### Layer 2 — Rate Calculation

Derived rates computed from Layer 1 data over a lookback window. Primary use case: [[Cost Per Km]] for vehicles, computed per bucket (`fuel`, `maintenance`, `tire`) — see [[ADR-016 — Cost Per Km Buckets]]. Rates are recalculated periodically (weekly/monthly), not per-query.

### Layer 3 — Per-Work-Order Cost Allocation

Uses Layer 1 (vehicle daily fixed costs, driver salary, per-trip + per-km rates, per-WO incidentals) and Layer 2 (variable-cost rates per bucket) to compute the cost of a specific [[Work Order]]:

```
work_order.cost =
    fixed_cost_share    -- vehicle fixed_time rules, daily share by distance ratio
  + variable_cost       -- Σ buckets × actual_distance_km
  + driver_cost         -- salary_share + per_trip + per_km × actual_distance
  + events_cost         -- per-WO one_off events with work_order_id
```

This is the operational view — used for pricing, route profitability, accept/decline decisions. See [[Integration — Order × Cost Ledger]].

## Key Design Principles

1. **Store facts, compute views.** The ledger stores what happened. Reports, dashboards, per-WO costs are all computed from the facts. See [[ADR-002 — Compute on Read, Cache When Needed]].

2. **Close and replace, never mutate.** Rules are time-versioned by close-and-replace; Events are append-only with reversing corrections. At most one open rule per `(entity_type, entity_id, cost_type)`. See [[ADR-001]] and [[ADR-017]].

3. **Classification drives views, not free-text labels.** `cost_classification` is the controlled enum that picks which view a cost lands in. `cost_type` is just a descriptive label. See [[ADR-014 — Cost Classification]].

4. **Month-by-month computation.** Date range queries split into month chunks because monthly proration depends on days-in-month. Feb (28 days) and Dec (31 days) have different daily rates for the same monthly salary. See [[ADR-003 — Month-by-Month Proration]].

5. **Vehicle purchase as depreciation rule.** A 1M vehicle purchase is not a cost spike — it's spread as a daily depreciation rule over the vehicle's expected life. See [[ADR-004 — Vehicle Purchase Depreciation Spread]].

6. **Per-WO incidentals are first-class.** Tolls, parking, port fees go in the same ledger via `work_order_id` on Cost Event. See [[ADR-015 — Per-WO Cost Events]].

7. **Driver salary by duration share.** Monthly retainer plugs into per-WO cost based on `duration_seconds / standard_working_seconds_per_day`, not by same-day count. See [[ADR-019 — Driver Salary Allocation by Duration Share]].

8. **Two reporting buckets, one ledger.** Per-WO direct cost (vehicle + driver + WO incidentals) and overhead ([[Monthly Operational Cost]]) are reporting distinctions on the same primitives. See [[ADR-013 — Driver as Cost Entity, Not Employee]].

9. **Managerial, not bookkeeping.** The ledger is accrual-basis and operational. The accountant's books are external and reconcile via export. See [[ADR-005 — Finance vs Operations Cost Views]].

## Key Decisions

- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-003 — Month-by-Month Proration]]
- [[ADR-004 — Vehicle Purchase Depreciation Spread]]
- [[ADR-005 — Finance vs Operations Cost Views]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-014 — Cost Classification]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-017 — Append-Only CostEvents]]
- [[ADR-018 — Negative Amounts and Disposal]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]

## Connects To

- **[[Model — Order to Work Order Chain]]** — the central data model showing how Cost Ledger feeds per-WO and per-OrderLine cost. Read first.
- [[Integration — Vehicle Maintenance × Cost Ledger]] — maintenance events as `variable_usage` Events; contracts as `fixed_time` Rules
- [[Integration — Fuel Log × Cost Ledger]] — fuel refills as `variable_usage` Events feeding the `fuel` bucket
- [[Integration — Driver × Cost Ledger]] — driver salary, per-trip, per-km
- [[Integration — User × Cost Ledger]] — office staff salary as `fixed_time` Rules in overhead bucket
- [[Integration — Order × Cost Ledger]] — per-WO cost composition and revenue/cost margin
- [[Monthly Operational Cost]] — the overhead bucket
- [[Vehicle Maintenance — Overview]] — source of maintenance spend data
- [[Driver Management — Overview]] — source of driver compensation