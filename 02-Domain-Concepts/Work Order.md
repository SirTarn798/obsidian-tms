# Work Order

#concept #module/work-order #status/active

> Assignment of one or more vehicles and one or more drivers to execute a set of stops on a given date. The unit of operations cost.

## Definition

A Work Order is the execution layer. It assigns real resources — a tractor (and optionally trailers), one main driver, and optionally helpers — to a set of [[Order Location]]s via [[Work Order Stop]]s. A single Work Order can cover stops from multiple [[Order]]s — VRP may merge routes for efficiency. Revenue belongs to Orders; cost belongs to the Work Order.

Vehicle assignment is captured in [[Work Order Vehicle]] (junction with role: tractor / trailer). See [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]]. Driver assignment is captured in [[Work Order Driver]] (junction with role: main / helper). See [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]].

## Key Fields

| Field | Type | Description |
|---|---|---|
| `work_order_id` | string | Unique identifier |
| `date` | date | Scheduled execution date |
| `status` | enum | `planned` → `in_progress` → `completed` / `closed_partial` / `cancelled` |
| `route_id` | string | Linked [[Route]] |
| `migrated_from_work_order_id` | string | If created via Migrate, source WO id (nullable) |
| `planned_start` | datetime | Operator-set or VRP-derived planned trip start. Used for overlap soft-block window before `driving_started_at` exists. See [[ADR-028]]. |
| `planned_end` | datetime | Operator-set or VRP-derived planned trip end. Used for overlap soft-block window before `work_ended_at` exists. See [[ADR-028]]. |
| `accepted_at` | datetime? | When the driver acknowledged the assignment |
| `driving_started_at` | datetime? | When the driver started driving — beginning of working time |
| `delivery_ended_at` | datetime? | When the last drop was completed — end of customer-facing work |
| `work_ended_at` | datetime? | When the driver returned to base or went off-duty — end of working time |
| `subcontract_actual_plate` | string? | Sub-contract WOs only: the actual tractor / rig plate the partner sent for this trip. See [[ADR-029]]. |
| `duration_seconds` | int (derived) | `work_ended_at − driving_started_at`. Used for driver salary share. |
| `odometer_start` | decimal | Tractor odometer at trip start (km) |
| `odometer_end` | decimal | Tractor odometer at trip end (km) |
| `total_distance_actual_km` | decimal | `odometer_end − odometer_start` — actual trip distance |
| `cost` | decimal (derived) | Total cost from [[Cost Ledger]] allocation, computed on read |
| `display_revenue` | decimal (derived) | Ops view only — computed on read, never invoiced |
| `display_margin` | decimal (derived) | `display_revenue − cost` — ops view only |

## Timestamp Lifecycle

Four timestamps mark the trip lifecycle. They are populated as the WO progresses and are nullable until reached.

```
WO created (status=planned)
  ↓
Driver acknowledges  →  accepted_at
  ↓
Driver starts driving  →  driving_started_at, status=in_progress
  ↓
... driver executes stops, odometer + evidence captured per [[Work Order Stop]] ...
  ↓
Last drop completed  →  delivery_ended_at
  ↓
Driver back to base / off-duty  →  work_ended_at, status=completed
```

`accepted_at` does not advance status — the WO remains `planned` until driving begins. `delivery_ended_at` is informational; the WO remains `in_progress` until `work_ended_at` is set.

## Date Anchor

`WorkOrder.date` is anchored on `driving_started_at::date` per [[ADR-033 — Work Order Date Anchored on driving_started_at]]. A trip that begins at 22:00 on Monday and finishes at 03:00 Tuesday has `date = Monday`. The anchor is locked at the `planned → in_progress` transition; if the planned date drifts before driving starts, the WO realigns to the actual start-day at that transition.

Single anchor — no splitting. Driver salary daily share, [[Monthly Operational Cost]] reports, fleet utilisation, and [[Driver Rate]] resolution all read this one date.

`cancelled` is reserved for WOs cancelled before `driving_started_at` is set (no work performed). Once driving has started, a WO that fails partially closes as `closed_partial` (with migration); a WO that fails completely after starting closes as `completed` with all stops in terminal non-`done` status.

`duration_seconds = work_ended_at − driving_started_at`. This is the working time used by [[ADR-019 — Driver Salary Allocation by Duration Share]] to allocate driver monthly salary to this WO. It excludes time before driving begins (waiting for assignment, pre-trip prep) and includes return-to-base time after the last drop.

For WOs in `in_progress` status, live dashboards may estimate duration from [[Route]] (`estimated_duration_min × 60`); the canonical value is set at WO close.

## Planned vs Actual Distance

| Field | Source | Purpose |
|---|---|---|
| `route.total_distance_km` | VRP / routing engine | Planned — used for cost estimation before trip |
| `work_order.total_distance_actual_km` | Odometer | Actual — used for final cost calculation after trip |

Cost allocation uses actual distance when available, planned distance as fallback.

## Display Revenue (Ops View)

`display_revenue` computed on read — not canonical:

```
display_revenue = Σ order_location.allocated_share
                  WHERE work_order_stop.work_order_id = this
                    AND work_order_stop.status = done
                    AND order_location.stop_type = drop
```

Used for route profitability dashboards. Never invoiced. See [[ADR-006 — Revenue Lives on Order, Not Work Order]].

## Cost

Computed from [[Cost Ledger]] using actual distance (or planned if trip not yet complete):

```
cost = fixed_cost_share + variable_cost + driver_cost + events_cost
```

| Component | Source |
|---|---|
| `fixed_cost_share` | Σ over [[Work Order Vehicle]] rows (owned/leased only) of each vehicle's `fixed_time` CostRules' daily share. See [[ADR-030]]. |
| `variable_cost` | Σ over [[Work Order Vehicle]] rows (owned/leased only): `(Σ applicable_cpk_buckets) × total_distance_actual_km`. Tractor buckets = fuel + maintenance + tire; trailer buckets = maintenance + tire. See [[ADR-016 — Cost Per Km Buckets]] and [[ADR-030]]. |
| `driver_cost` | Sum across [[Work Order Driver]] rows: duration-share salary + per-trip + per-km, each resolved against [[Driver Rate]] |
| `events_cost` | `Σ CostEvents WHERE work_order_id = this.id` per [[ADR-015 — Per-WO Cost Events]] |

For sub-contract WOs (assigned vehicle has `ownership = sub_contracted`) the cost formula short-circuits to `subcontract_payout` CostEvent + actual `events_cost`. Fixed/variable/driver terms skipped. See [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]].

Per-stop cost on [[Work Order Stop]] is derived from these four components per [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]] and rolls back up the Order chain.

See [[Integration — Driver × Cost Ledger]] and [[Integration — Order × Cost Ledger]].

## Migrate

When a Work Order must be abandoned mid-trip:
1. Close as `closed_partial`
2. Mark remaining WO Stops as `migrated`
3. Create new WO with new vehicle and new [[Work Order Driver]] rows
4. Link via `migrated_from_work_order_id`

See [[ADR-010 — Work Order Migration on Failure]].

Migration is also the path for **main driver swap** or **vehicle change** mid-trip. **Helper changes** mid-trip stay on the same WO and edit the [[Work Order Driver]] junction directly per [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]].

## Overlap Soft-Block (Vehicle)

Per [[ADR-028 — Driver and Vehicle Overlap Soft-Block]] and [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]]: assigning a vehicle that overlaps another non-cancelled WO's effective window triggers a soft-block. The check applies per-asset — tractor and each trailer independently. Override fields live on the [[Work Order Vehicle]] row that triggered the soft-block.

Sub-contract slots (Vehicle with `ownership = sub_contracted`) are excluded from the check per [[ADR-029]] — partner manages capacity allocation.

Effective window for overlap math:

```
window_start = driving_started_at  IF status IN (in_progress, completed, closed_partial)
               ELSE planned_start
window_end   = work_ended_at       IF status = completed | closed_partial
               ELSE planned_end    (or live estimate if in_progress)
```

Cancelled WOs are excluded from the check.

## Related

- [[Work Order Vehicle]] — tractor + trailer assignments (junction)
- [[Work Order Driver]] — main + helper assignments
- [[Work Order Stop]] — joins to Order Locations, holds per-stop mileage + evidence
- [[Stop Evidence]] — proof of delivery (via Work Order Stop)
- [[Route]] — planned path
- [[Order]] — revenue source (via stops → Order Locations → Order Lines → Orders)
- [[Vehicle]] · [[Driver]] · [[Subcontractor]]
- [[Integration — Order × Work Order]]
- [[Integration — Order × Cost Ledger]]
- [[Integration — Driver × Work Order]]
- [[Integration — Driver × Cost Ledger]]
- [[Model — Order to Work Order Chain]]
- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
- [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]]
- [[ADR-033 — Work Order Date Anchored on driving_started_at]]
