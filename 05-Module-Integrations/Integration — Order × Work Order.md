# Integration — Order × Work Order

#integration #module/order-mgmt #module/work-order #status/active

> How Order stops flow to Work Order execution, and how VRP re-planning, migration, and completion propagate back.

## Overview

[[Order]]s declare what needs to be done (via [[Order Line]]s and [[Order Location]]s). [[Work Order]]s declare who does it and when. The [[Work Order Stop]] join table is the bridge — movable and ephemeral until executed.

## Full Chain

```
Order
  └── OrderLine
        └── OrderLocation (stop intent)
                  ↓ (VRP assigns)
            Work Order Stop (join: WO ↔ OL)
                  ↑
            Work Order (vehicle, driver, date)
                  ↑
            Route (path, distance)
```

## VRP Assignment Flow

```
1. Confirmed OrderLocations available for planning (status=planned, no WO Stop yet)
2. VRP runs across all pending OrderLocations
3. For each optimised route:
   a. Create Work Order (vehicle, driver, date)
   b. Create Route (polyline, total_distance_km)
   c. Create Work Order Stops (one per OrderLocation, sequence + planned_arrival)
4. OrderLocations now locked (WO Stop exists)
5. Parent Orders → status=in_progress
```

## Re-plan (before execution)

```
1. VRP re-runs with updated constraints
2. Delete Work Order Stops for unexecuted OrderLocations
3. Drop empty Work Orders (no stops remaining)
4. Create new Work Orders + Routes + WO Stops
5. OrderLocations: untouched
6. OrderLines: untouched
7. Order revenue: untouched
```

## Stop Completion (back-propagation)

```
Driver marks stop done:
  Work Order Stop.status = done
  Work Order Stop.actual_arrival = now
  Order Location.status = done

When all OrderLocations for an Order are terminal (done/cancelled/failed):
  Order checks effective_invoice_policy across all OrderLines
  order.billable_revenue computed
  order.status = completed (auto) OR waits for manual close
```

## Migration (mid-trip failure)

```
Remaining WO Stops → status = migrated
Old Work Order    → status = closed_partial

New Work Order created:
  migrated_from_work_order_id = old WO
  New WO Stops: same OrderLocations, status = planned
  OrderLocation statuses reset: migrated → planned
  New Route computed
```

See [[ADR-010 — Work Order Migration on Failure]].

## Display Revenue Computation

```
WO.display_revenue = Σ order_location.allocated_share
                     WHERE wos.work_order_id = this WO
                       AND wos.status = done
                       AND ol.stop_type = drop

(allocated_share comes from OrderLine's AllocationRule)
```

Ops view only. Never invoiced. See [[ADR-006 — Revenue Lives on Order, Not Work Order]].

## What Each Layer Owns

| Concern | Owner |
|---|---|
| Which services customer ordered | Order + OrderLines |
| Agreed price per service | OrderLine (PriceRule snapshot) |
| Revenue per stop for display | OrderLocation (allocated_share) |
| Who drives, what date | Work Order |
| Route geometry | Route |
| Stop sequence + actual timing | Work Order Stop |
| Cost of the trip | Work Order |
| Customer invoice amount | Order.billable_revenue |

## Related

- [[Order]]
- [[Order Line]]
- [[Order Location]]
- [[Work Order]]
- [[Work Order Stop]]
- [[Route]]
- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-010 — Work Order Migration on Failure]]
