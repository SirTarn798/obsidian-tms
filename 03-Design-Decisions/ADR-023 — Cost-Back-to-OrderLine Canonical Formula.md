# ADR-023 — Cost-Back-to-OrderLine Canonical Formula

#decision #module/order-mgmt #module/work-order #module/cost-ledger #status/active

## Status

Accepted. Locks the cost flow that [[Model — Order to Work Order Chain]] sketches.

## Context

[[Work Order]] is the unit of operational cost. [[Cost Ledger]] composes WO cost from four components — fixed share, variable, driver, events. But the customer commitment lives on [[Order Line]] / [[Order Location]], and per-route P&L queries on the Order side need cost just as much as revenue.

The traceability problem: VRP merges 6 OrderLines into one WorkOrder, then a single split moves three of them to a different vehicle next morning. To answer "what did Customer A's BKK→CNX run cost us this month," cost must derive back through the WorkOrderStop join to the OrderLocation and OrderLine — and this derivation must not depend on which WO they happened to land on.

The flow needs:

1. A **per-stop** cost number on [[Work Order Stop]] so each OrderLocation can read a single value.
2. A **defensible split** of the four WO-level components down to per-stop level.
3. A simple aggregation rule from OrderLocation up to OrderLine and Order.
4. **Compute on read**: cost is derived, never stored as a fact (matches [[ADR-002 — Compute on Read, Cache When Needed]]).

## Decision

**Cost flows from WorkOrder down to WorkOrderStop, then back up the Order chain through the WorkOrderStop ↔ OrderLocation join.**

### Per-WorkOrderStop split

Let `WO` be a WorkOrder with `N = count(WorkOrderStop)`.

```
wos.variable_cost  = (fuel_cpk + maintenance_cpk + tire_cpk)
                   × wos.distance_actual_km

wos.fixed_share    = WO.fixed_cost_share / N
wos.driver_share   = WO.driver_cost      / N
wos.events_share   = WO.events_cost      / N

wos.cost = wos.variable_cost
         + wos.fixed_share
         + wos.driver_share
         + wos.events_share
```

| Component | Allocation | Why |
|---|---|---|
| `variable_cost` | Per-stop leg distance | Each stop's actual leg distance is known from the WorkOrderStop odometer; matches physical fuel/maintenance consumption. |
| `fixed_cost_share` | Even split by stop count | Vehicle daily depreciation/insurance is paid for the trip, not the kilometres — distance is already covered in `cost_per_km`. Splitting by stops is simple and defensible; refining to "by stop time" is held for v2. |
| `driver_cost` | Even split by stop count | Driver salary share is already proportional to WO duration via [[ADR-019 — Driver Salary Allocation by Duration Share]]. Per-stop refinement would need per-stop time tracking — held for v2. |
| `events_cost` | Even split by stop count | Default. Customer-specific extras tied to one stop can be refined later with an optional `order_location_id` on [[Cost Event]] — held for v2. |

The four WO-level components themselves are computed per [[Integration — Order × Cost Ledger]]:

```
WO.fixed_cost_share = Σ (vehicle fixed_time CostRules' daily share, by distance ratio)
WO.variable_cost    = Σ (cpk_bucket × WO.distance_actual_km)
WO.driver_cost      = duration_based_salary + per_trip + (per_km × distance_actual)
WO.events_cost      = Σ (CostEvents WHERE work_order_id = WO.id)
```

`WO.cost = fixed_cost_share + variable_cost + driver_cost + events_cost`.

Per-stop variable cost as defined above sums to `WO.variable_cost` exactly because `Σ wos.distance_actual_km = WO.distance_actual_km`. The other three components also sum to their WO totals because the splits are exhaustive.

### Roll-up

```
OrderLocation.cost = Σ wos.cost
                     WHERE wos.order_location_id = this.id
                     -- usually one row; multiple rows when migration occurred
                     -- (old wos status=migrated, new wos status=done)

OrderLine.cost     = Σ OrderLocation.cost             -- across this line's stops
Order.cost         = Σ OrderLine.cost
```

In the no-migration case, the OrderLocation has exactly one WorkOrderStop and the sum is degenerate. Migration ([[ADR-010 — Work Order Migration on Failure]]) is the only path that produces multiple WorkOrderStops per OrderLocation; both the migrated leg and the completing leg cost real money and both flow back to the OrderLocation.

For pickups: `OrderLocation.cost` includes the leg cost regardless of stop type. Allocation of revenue is drops-only ([[Allocation Rule]]); cost allocation is per-stop because each stop genuinely consumes road.

### Compute-on-read

All `*.cost` fields above are derived. The persisted facts are CostRules, CostEvents, WorkOrderStop.distance_actual_km, and the WO timestamps. WO/OrderLine/Order cost queries recompute from these facts. Caching policy follows [[ADR-002 — Compute on Read, Cache When Needed]] — finalise on WO close, recompute if input facts change.

### Migration

When a WorkOrder migrates ([[ADR-010 — Work Order Migration on Failure]]), each migrated WorkOrderStop becomes terminal on the old WO and a new WorkOrderStop on the new WO carries forward. Cost on the old WorkOrderStop reflects what was incurred up to that point under the old WO; cost on the new WorkOrderStop reflects the new WO's components. The roll-up formula above already sums across all WorkOrderStops referencing the OrderLocation, so migration is handled with no special case.

### Trace stability

VRP merge/split changes which `WO` a `WorkOrderStop` belongs to. It does not change `OrderLocation.cost` semantics — the join is still through WorkOrderStops, and per-stop cost is computed from the components of whichever WO the stop ended up on. `Σ OrderLine.cost WHERE service_id = X AND order.customer = A` is well-defined regardless of merges.

## Consequences

- **Good:** Per-OrderLine and per-Order cost are well-defined and VRP-stable.
- **Good:** Per-WO total is preserved exactly when summing WorkOrderStop costs.
- **Good:** Margin per OrderLine = `revenue − cost` with both sides independently derivable.
- **Good:** Compute-on-read means VRP re-plans, CostRule close-and-replace, and CostEvent append-only corrections all flow through naturally.
- **Tradeoff:** Even split for fixed/driver/events is approximate. A 30-minute stop and an 8-hour stop on the same WO get the same fixed share. Refinement deferred to v2.
- **Tradeoff:** Cost queries join through WorkOrderStops to OrderLocations. The query path is well-indexed but non-trivial.

## Related

- [[Order]]
- [[Order Line]]
- [[Order Location]]
- [[Work Order]]
- [[Work Order Stop]]
- [[Cost Ledger — Overview]]
- [[Integration — Order × Cost Ledger]]
- [[Model — Order to Work Order Chain]]
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
