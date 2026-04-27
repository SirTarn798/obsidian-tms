# Work Order — Overview

#module/work-order #status/active

> Assigns vehicle and driver to execute order stops. The unit of operational cost. Serves the Order layer.

## Context

Work Order Management answers: "Who does what, when, and how much did it cost us?" It is the execution layer. VRP optimises Work Orders freely; the Order layer above it is stable.

Work Orders are intentionally volatile — they can be created, re-planned, split, merged, and migrated without affecting customer-facing data. Cost belongs here. Revenue belongs to [[Order]].

## How It Works

### Layer 1 — Assignment

VRP (or manual assignment) creates a [[Work Order]]:
- Vehicle assigned
- Driver assigned
- Date set
- [[Route]] computed (polyline, distance, estimated timing)

### Layer 2 — Stop Assignment

[[Work Order Stop]]s are created: one row per [[Order Location]] assigned to this Work Order.

A single Work Order can hold stops from multiple Orders (VRP merges for efficiency).

### Layer 3 — Execution

Driver completes stops in sequence. Each [[Work Order Stop]] status updated: `planned → done | failed | cancelled`.

Corresponding [[Order Location]] status synced.

### Layer 4 — Cost Allocation

At Work Order close, cost computed from [[Cost Ledger]]:

```
work_order.cost =
    fixed_cost_share (vehicle depreciation + insurance, proportional to WO distance)
  + variable_cost (cost_per_km × route.total_distance_km)
  + driver_cost (driver rate × trip duration)
```

See [[Integration — Order × Cost Ledger]].

### Layer 5 — Display Revenue (Ops View)

`display_revenue` computed on read. Not stored canonically.

```
display_revenue = Σ order_location.allocated_share
                  WHERE wos.work_order_id = this
                    AND wos.status = done
                    AND ol.stop_type = drop
```

`display_margin = display_revenue − cost`. Used for route profitability. Never invoiced.

## VRP Re-plan

Before any stop is executed:
- Delete old [[Work Order Stop]] rows
- Create new rows (same Order Locations, new Work Orders / sequences)
- Create new Routes
- Orders untouched

After stops executed: those WO Stop rows are permanent, immutable.

## Migration (Mid-trip Failure)

See [[ADR-010 — Work Order Migration on Failure]].

1. Close current WO (`closed_partial`)
2. Mark remaining WO Stops as `migrated`
3. Create new WO with new vehicle/driver
4. New WO Stops created for migrated Order Locations
5. New Route computed
6. `migrated_from_work_order_id` links chain for audit

## Key Design Principles

1. **Cost lives here. Revenue lives on Order.**
2. **WO Stops are movable.** Order Locations are not.
3. **Close and replace on failure.** Never mutate a WO that has done stops.
4. **display_revenue is ops analytics.** Never use for finance.

## Key Decisions

- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-010 — Work Order Migration on Failure]]

## Connects To

- **[[Model — Order to Work Order Chain]]** — the central data model showing how WorkOrder serves the Order layer above. Read first.
- [[Integration — Order × Work Order]] — how stops flow from Order layer
- [[Integration — Order × Cost Ledger]] — cost calculation for this WO
- [[Order Management — Overview]] — the layer this serves
- [[Cost Ledger — Overview]] — source of cost rates

## Open Questions

- Should Work Order track idle time (driver waiting at customer site) separately from travel time?
- How to handle toll costs — per-WO event, or rolled into variable cost?
- Driver overtime — cost rule or separate event?
