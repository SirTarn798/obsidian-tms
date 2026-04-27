# Work Order Driver

#concept #module/work-order #module/driver-mgmt #status/active

> Junction between [[Work Order]] and [[Driver]] that carries role and per-row time bounds. Replaces the single-driver field on Work Order.

## Definition

A Work Order may have **one main driver** and **zero or more helpers**. The Work Order Driver row captures each (Work Order, Driver) pair with its role and the time window during which that driver was on the trip. This model supersedes the previous `Work Order.driver_id` single-FK design — see [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]].

Helpers are paid the same way drivers are, just at helper rates ([[Driver Rate]]). Both main and helpers contribute to the Work Order's driver cost. Per-helper duration share is computed from the row's `added_at` / `removed_at` per [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]].

## Key Fields

| Field | Type | Description |
|---|---|---|
| `work_order_driver_id` | string | PK |
| `work_order_id` | string | Linked [[Work Order]] |
| `driver_id` | string | Linked [[Driver]] |
| `role` | enum | `main` / `helper` |
| `assigned_at` | datetime | Audit — when the row was created |
| `added_at` | datetime | When the driver effectively joins the trip. Equals `assigned_at` for rows added during planning; set to `now()` for mid-trip helper additions. Used for per-helper duration share. |
| `removed_at` | datetime? | NULL while driver is still on the trip. Set when a helper is released mid-trip. Removed helpers stay in the table for cost and audit. Always NULL for `main`. |
| `license_override_by_user_id` | string? | Set when assignment overrode the licence soft-block. See [[ADR-026]]. |
| `license_override_at` | datetime? | |
| `license_override_reason` | string? | Free text. Required when override fires. |
| `overlap_override_by_user_id` | string? | Set when assignment overrode a driver-overlap soft-block. See [[ADR-028]]. |
| `overlap_override_at` | datetime? | |
| `overlap_override_reason` | string? | Free text. Required when override fires. |

**Unique constraint:** `(work_order_id, driver_id)` — one driver may not appear twice on the same WO. Once `removed_at` is set, the row is immutable.

## Constraints

- Exactly one row per Work Order has `role = main` (where `removed_at IS NULL`).
- Helpers are unbounded — zero or more.
- Rows can be inserted/deleted while `WorkOrder.status = planned`.
- Once `WorkOrder.status` advances to `in_progress`:
  - `main` rows are **frozen**. Main change goes through migration ([[ADR-010 — Work Order Migration on Failure]]).
  - `helper` rows remain **mutable** — add/remove freely. Removal sets `removed_at`; the row stays. See [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]].
- All rows immutable once `WorkOrder.status = completed | closed_partial | cancelled`.

## Soft-Block Checks at Assignment

Two checks fire when inserting or updating a row. Both follow the soft-block + free-text override pattern:

### Licence (main only)

Per [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]:

```
allowed = driver.license_expiry_date >= work_order.date
       AND (driver.license_classes ∩ vehicle_type.required_license_classes) ≠ ∅

if not allowed:
  show warning naming the failed condition
  require "Proceed anyway" + non-empty reason
  set license_override_* fields
```

Helpers are not licence-checked.

### Overlap (any role)

Per [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]:

```
conflicts = WOs whose effective window overlaps this WO's window
            AND that have any non-cancelled WorkOrderDriver row for this driver
            (excluding this WO and excluding rows where removed_at <= wo.window_start)

if conflicts non-empty:
  show conflict list
  require "Proceed anyway" + non-empty reason
  set overlap_override_* fields
```

Window definition is in [[ADR-028]]. Both override field sets can be set on the same row independently.

## Cost Implication

Per Work Order:

```
work_order.driver_cost =
    Σ over WorkOrderDriver rows:
        salary_share(driver, wo, this_row)
      + per_trip_rate(driver, role, customer, service)   (resolved per-stop per ADR-025)
      + per_km_rate (driver, role, customer, service) × wo.total_distance_actual_km

salary_share for main row    = full WO duration share (per ADR-019)
salary_share for helper row  = (helper_duration_seconds / standard_working_seconds_per_day)
                             × DriverSalary.daily_share(wo.date)
```

Per-helper `helper_duration_seconds` defined in [[ADR-027]]. See also [[Integration — Driver × Cost Ledger]].

## Lifecycle

```
WO created (status=planned)
  → INSERT WorkOrderDriver(wo, main_driver, role=main, assigned_at, added_at = same)
    → licence soft-block fires; overlap soft-block fires
  → optionally INSERT helpers
    → overlap soft-block fires per helper

WO planning edits
  → INSERT/DELETE helpers freely
  → main may be swapped via DELETE+INSERT

WO starts (status=in_progress)
  → main row frozen
  → helpers still mutable:
      add helper:    INSERT row with added_at = now()  (overlap check fires)
      release helper: UPDATE existing row, removed_at = now()
      swap helper:    UPDATE outgoing removed_at + INSERT new row

Mid-trip main-change or vehicle-change → migration (ADR-010), not in-place

WO completed
  → all rows immutable
  → cost computed using junction rows + DriverRate resolution + per-row durations
```

## Mobile App Visibility

Both main and helpers see the Work Order in their mobile app feed for as long as their row exists with `removed_at IS NULL`. A released helper loses the WO from their live feed but keeps it in their history. Mutating actions (status updates, evidence, odometer) are restricted to `main`. Submitting [[Fuel Log]] is open to either role while the row is active.

## Related

- [[Work Order]]
- [[Driver]]
- [[Driver Rate]]
- [[Driver Salary]]
- [[Vehicle Type]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — Driver × Work Order]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-025 — DriverRate route_id Renamed to service_id]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]
- [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]
