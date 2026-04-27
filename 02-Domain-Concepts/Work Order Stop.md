# Work Order Stop

#concept #module/work-order #status/active

> Join between a [[Work Order]] and an [[Order Location]]. The movable link that VRP re-plans without touching orders. Records execution data and mileage per stop.

## Definition

Work Order Stop is the critical join in the execution layer. Records which [[Order Location]] a [[Work Order]] is responsible for, in what sequence, and what actually happened — including per-stop odometer readings and any [[Stop Evidence]].

[[Order Location]]s never move between orders. Work Order Stops can be created, deleted, or reassigned freely — this is how VRP re-planning and [[Work Order]] migration operate without corrupting order data.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `work_order_stop_id` | string | Unique identifier |
| `work_order_id` | string | Parent Work Order |
| `order_location_id` | string | The Order Location being executed |
| `sequence` | integer | Position in this WO's execution route |
| `planned_arrival` | datetime | VRP-estimated arrival time |
| `actual_arrival` | datetime | Recorded on completion |
| `status` | enum | `planned` → `done` / `failed` / `cancelled` / `migrated` |
| `odometer_start` | decimal | Vehicle odometer reading on departure to this stop (km) |
| `odometer_end` | decimal | Vehicle odometer reading on arrival at this stop (km) |
| `distance_actual_km` | decimal | `odometer_end − odometer_start` — actual leg distance |
| `notes` | string | Driver notes for this stop |
| `cost` | decimal | **Derived.** Per-stop share of the WO cost components. Computed on read; cached at WO close. |

## Stop Evidence

Each stop can have N [[Stop Evidence]] records — photos, signatures, documents, notes. Captured by driver app at time of delivery.

```
WorkOrderStop 1 ──→ N StopEvidence
```

## Mileage

`distance_actual_km` per stop enables:
- Actual vs planned distance comparison per leg
- Per-stop mileage for distance-based [[Allocation Rule]] (`by_distance`)
- Dispute resolution (did driver take a detour?)
- Per-stop variable cost in the cost-back formula

Trip-level actual mileage lives on [[Work Order]] (`odometer_start`, `odometer_end`, `total_distance_actual_km`).

## Cost (Derived)

```
wos.variable_cost = (fuel_cpk + maintenance_cpk + tire_cpk) × wos.distance_actual_km

N = count(WorkOrderStops in this WO)
wos.fixed_share   = WO.fixed_cost_share / N
wos.driver_share  = WO.driver_cost      / N
wos.events_share  = WO.events_cost      / N

wos.cost = wos.variable_cost + wos.fixed_share + wos.driver_share + wos.events_share
```

Variable cost is per-stop by leg distance; fixed / driver / events are split evenly by stop count. Per [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]. The four WO-level components are computed per [[Integration — Order × Cost Ledger]].

`wos.cost` rolls up to [[Order Location]] (sum across all WorkOrderStops referencing the same OrderLocation, which handles migration without special cases) and on through OrderLine and Order.

## Status Sync

When WO Stop status changes → corresponding [[Order Location]] status updated to match.

`migrated` is terminal — stop transferred to new WO. Original row preserved for audit.

## VRP Re-plan

```
Re-plan before departure:
  Delete old WO Stops for unexecuted OrderLocations
  Insert new WO Stops (same Order Locations, new Work Orders/sequences)
  Order Locations: untouched. Order revenue: untouched.
```

## Migrate (mid-trip failure)

```
Completed stops: status=done — permanent, immutable
Remaining stops: status=migrated (terminal on old WO)
New WO: new WO Stop rows for same OrderLocations, status=planned
```

See [[ADR-010 — Work Order Migration on Failure]].

## Related

- [[Work Order]] — parent
- [[Order Location]] — the stop being executed
- [[Stop Evidence]] — proof attached to this stop
- [[Cost Ledger — Overview]] — source of cost components
- [[Model — Order to Work Order Chain]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
