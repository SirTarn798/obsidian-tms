# Integration — Driver × Work Order

#integration #module/driver-mgmt #module/work-order #status/active

> How [[Driver]]s are assigned to [[Work Order]]s, what license rules apply, and how the assignment is scoped on the mobile app.

## Overview

A Work Order has one **main driver** and any number of **helpers**. The relationship is captured in [[Work Order Driver]] — see [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]. The previous single `Work Order.driver_id` field is removed.

Drivers see their assigned Work Orders through the mobile app, scoped by the `driver_id` claim in their JWT.

## Assignment

### When WO is created

```
1. Operator creates Work Order (status=planned)
2. Operator picks vehicle → Vehicle Type known
   → vehicle-overlap soft-block check fires
3. Operator picks main driver
   → main driver dropdown filters by:
       license_expiry_date >= wo.date
       AND license_classes ∩ vehicle_type.required_license_classes ≠ ∅
   → "Show all drivers" toggle exposes the full list
   → on selection: licence soft-block fires; driver-overlap soft-block fires
4. Operator optionally picks helpers (zero or more)
   → each helper: driver-overlap soft-block fires
5. INSERT WorkOrderDriver rows, one per assignment, with role + assigned_at + added_at
```

### Soft-Block Pattern

All three soft-blocks (licence, driver-overlap, vehicle-overlap) follow the same pattern:

```
detect violation
  → show warning describing the failed condition
  → require explicit "Proceed anyway" click
  → require non-empty free-text reason
  → record override_*_by_user_id, override_*_at, override_*_reason on:
        WorkOrderDriver  for licence + driver-overlap
        WorkOrder        for vehicle-overlap
```

Override rows feed compliance reports; high override frequency is the operational signal that something needs fixing (driver record stale, roster too thin, controller training needed).

### Licence Soft-Block (main only)

Per [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]:

```
on assign(driver, role=main, wo):
  vehicle_type = wo.vehicle.vehicle_type
  expiry_ok    = driver.license_expiry_date IS NOT NULL
                 AND driver.license_expiry_date >= wo.date
  class_ok     = (driver.license_classes ∩ vehicle_type.required_license_classes) ≠ ∅

  if not (expiry_ok AND class_ok):
      show warning naming the failed condition (expired vs. missing class)
      require "Proceed anyway" + non-empty reason
      set license_override_* on the WorkOrderDriver row
```

Helpers are not licence-checked.

### Overlap Soft-Block (any role; vehicle separately)

Per [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]:

```
driver_conflicts = WOs whose effective window overlaps wo.window
                   AND that have any non-cancelled WorkOrderDriver row
                       for this driver (excluding this WO,
                       excluding rows where removed_at <= wo.window_start)

vehicle_conflicts = WOs whose effective window overlaps wo.window
                    AND vehicle_id = wo.vehicle_id
                    AND status != 'cancelled'

if driver_conflicts non-empty:
    show conflict list, require override on the WorkOrderDriver row
if vehicle_conflicts non-empty:
    show conflict list, require override on the WorkOrder row
```

`window_start` / `window_end` definition is in [[ADR-028]] — `driving_started_at` / `work_ended_at` when set, else `planned_start` / `planned_end`.

### Edits While Planned

While `WorkOrder.status = planned`:

- Helpers can be added or removed at any time.
- Main can be swapped (DELETE old main row, INSERT new main row).
- Each insert re-runs all applicable soft-block checks.

### Edits While In-Progress (per ADR-027)

Once `WorkOrder.status = in_progress`, the rules diverge by role:

| Action | Allowed? | Notes |
|---|---|---|
| Add helper | Yes | INSERT row with `added_at = now()`. Overlap check fires. |
| Remove helper | Yes | UPDATE existing row, `removed_at = now()`. Row stays for cost + audit. |
| Swap helper | Yes | UPDATE outgoing `removed_at` + INSERT new with `added_at = now()`. |
| Change main | No → use migration ([[ADR-010]]) | |
| Change vehicle | No → use migration | |

Once `WorkOrder.status = completed | closed_partial | cancelled`, all rows are immutable.

### Promote Helper to Main

If main becomes incapacitated mid-trip and a helper holds the appropriate licence, the path is **migration**, not in-place promotion. Close current WO as `closed_partial`; create a new WO with the helper as `main` (licence check fires); remaining helpers are added to the new WO with fresh `added_at`. Per [[ADR-027]].

## Mobile App Scope

Once a driver logs into the mobile app, the JWT carries `driver_id`. All driver-facing endpoints filter by it. Released helpers (`removed_at IS NOT NULL`) drop out of the live feed but remain in history.

```
GET /mobile/work_orders/today
  → SELECT wo.* FROM WorkOrder wo
    JOIN WorkOrderDriver wod ON wod.work_order_id = wo.work_order_id
    WHERE wod.driver_id    = jwt.driver_id
      AND wod.removed_at  IS NULL
      AND wo.date          = today
    ORDER BY wo.date, wo.start_time

GET /mobile/work_orders/{wo_id}
  → returns WO detail only if EXISTS WorkOrderDriver(wo_id, jwt.driver_id)
    with removed_at IS NULL (live) OR for history view
  → 403 otherwise
```

The driver sees the WO regardless of role. The response includes `my_role` derived from the junction row.

## Status Transitions Driver Can Trigger

From the mobile app, the driver may:

| Action | Permitted role | Effect |
|---|---|---|
| Set odometer_start | main only | `WorkOrder.odometer_start` |
| Update stop status (arrived/done/failed) | main only | `WorkOrderStop.status` |
| Upload [[Stop Evidence]] | main only | Photo / signature on done stop |
| Set odometer_end | main only | `WorkOrder.odometer_end`; computes `total_distance_actual_km` |
| Submit [[Fuel Log]] | any (main or helper) | New FuelLog + Cost Event on vehicle |

Helpers see the WO in their list but cannot mutate trip state. Their accrual records show automatically based on the WO's main-driver-driven status.

## Cost Calculation Hook

When a Work Order completes:

```
on WorkOrder.status → completed | closed_partial:
  for each row in WorkOrderDriver(wo):
     if role = main:
        salary_share = full WO duration share (per ADR-019)
     else:                                   -- helper
        helper_duration = max(0,
            min(removed_at OR work_ended_at, work_ended_at)
          - max(added_at,                    driving_started_at))
        salary_share = DriverSalary.daily_share(wo.date)
                     × (helper_duration / standard_working_seconds_per_day)

     resolve per_trip and per_km via DriverRate (per-stop per ADR-025)
     accrue per-driver compensation

  work_order.driver_cost = Σ accruals
  work_order.cost = fixed_cost_share + variable_cost + driver_cost + events_cost
```

See [[Integration — Driver × Cost Ledger]] for resolution detail and [[Integration — Order × Cost Ledger]] for the WO total cost formula.

## API Surface (Web — Controller)

```
POST /work_order/{wo_id}/drivers
  { driver_id, role: main | helper,
    license_override_reason?,         -- required if licence soft-block fires
    overlap_override_reason? }        -- required if overlap soft-block fires
  → INSERT WorkOrderDriver
  → fires licence soft-block if role=main
  → fires driver-overlap soft-block (any role)

DELETE /work_order/{wo_id}/drivers/{driver_id}
  → if WO.status = planned: hard DELETE
  → if WO.status = in_progress AND role = helper: UPDATE removed_at = now()
  → otherwise: 409

PATCH /work_order/{wo_id}/vehicle
  { vehicle_id, vehicle_overlap_override_reason? }
  → only allowed while WO.status = planned
  → fires vehicle-overlap soft-block

GET /work_order/{wo_id}/drivers
  → list active (removed_at IS NULL) + ?include_history=true for the full set
```

## Related

- [[Driver]]
- [[Vehicle Type]]
- [[Work Order]]
- [[Work Order Driver]]
- [[Driver Rate]]
- [[Driver Salary]]
- [[Stop Evidence]]
- [[Fuel Log]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — Order × Cost Ledger]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-025 — DriverRate route_id Renamed to service_id]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]
- [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]
