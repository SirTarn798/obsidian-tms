
> Event-sourced cost tracking system that records what happened and when, then computes any cost view on demand.

#module/cost-ledger #status/active

## Context

The platform needs to track costs across multiple entity types ([[Vehicle]], [[Employee]], future: facilities, equipment, etc.) with full historical accuracy. Costs are messy in practice — employees join mid-month, vehicles are purchased on arbitrary dates, salaries change, maintenance is lumpy. We need a system that handles all of this without special-casing each scenario.

## How It Works

The system has three layers:

### Layer 1 — Ledger (Source of Truth)

Two primitives store everything:

- **[[Cost Rule]]** — An ongoing cost obligation with a time range. "From date X to date Y, this entity costs Z per period." Rules are never mutated — they are closed and replaced. See [[ADR-001 — Close and Replace, Never Mutate]].
    
- **[[Cost Event]]** — A one-time cost at a point in time. Purchases, repairs, bonuses — things that happened once.
    

### Layer 2 — Rate Calculation

Derived rates computed from Layer 1 data over a look back window. Primary use case: [[Cost Per Km]] for vehicles, calculated from maintenance history and odometer readings. Rates are recalculated periodically (weekly/monthly), not per-query.

### Layer 3 — Delivery Cost Allocation

Uses Layer 1 (daily fixed costs) and Layer 2 (cost/km rates) to compute the cost of a specific [[Delivery]]. This is the operational view — used for pricing, route profitability, deciding whether to accept a job.

See [[Integration — Cost Ledger × Delivery Costing]] for full details.

## Key Design Principles

1. **Store facts, compute views.** The ledger stores what happened. Reports, dashboards, delivery costs are all computed from the facts. See [[ADR-002 — Compute on Read, Cache When Needed]].
    
2. **Month-by-month computation.** Any date range query is split into month chunks because monthly proration depends on days-in-month. Feb (28 days) and Dec (31 days) have different daily rates for the same monthly salary. See [[ADR-003 — Month-by-Month Proration]].
    
3. **Vehicle purchase as depreciation rule.** A 1M vehicle purchase is not a cost spike — it's spread as a daily depreciation rule over the vehicle's expected life. See [[ADR-004 — Vehicle Purchase Depreciation Spread]].
    
4. **Finance vs Operations are different views.** Monthly finance reports use actual spend (matches bank account). Delivery costing uses smoothed rates (matches operational reality). See [[ADR-005 — Finance vs Operations Cost Views]].
    

## Key Decisions

- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-003 — Month-by-Month Proration]]
- [[ADR-004 — Vehicle Purchase Depreciation Spread]]
- [[ADR-005 — Finance vs Operations Cost Views]]

## Connects To

- [[Integration — Vehicle Maintenance × Cost Ledger]] — maintenance events feed into the ledger as CostEvents, maintenance contracts as CostRules
- [[Integration — Employee Management × Cost Ledger]] — hiring, salary changes, termination create/close CostRules
- [[Integration — Cost Ledger × Delivery Costing]] — delivery cost = fixed allocation + variable (cost/km × distance)
- [[Vehicle Maintenance — Overview]] — source of maintenance spend data
- [[Employee Management — Overview]] — source of salary/compensation data

## Open Questions

- How to handle fuel costs — per-vehicle rule (monthly estimate) or per-delivery event (actual fuel used)?
- Should depreciation method be configurable per vehicle (straight-line vs declining balance)?
- How to handle vehicle disposal/sale — create a negative cost event for residual value?