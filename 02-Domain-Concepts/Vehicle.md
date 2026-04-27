# Vehicle

#concept #module/work-order #module/cost-ledger #status/active

> A specific operating asset — owned, leased, or a partner-supplied supply slot. Referenced by [[Work Order]] via the [[Work Order Vehicle]] junction.

## Definition

A Vehicle is a row representing one operating asset. For owned and leased vehicles, the row is a specific physical truck (plate, vehicle type, odometer, lifecycle, maintenance history). For sub-contracted vehicles, the row is a **supply slot** — "Partner X can supply a 22-wheel on demand" — used across many trips. The actual plate the partner sends each trip is captured per-trip on [[Work Order]], not by creating new Vehicle rows.

This model is locked by [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]] and complemented by [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]] for multi-asset assignments.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `vehicle_id` | string | PK |
| `vehicle_type_id` | string | FK to [[Vehicle Type]] |
| `plate_number` | string? | Required iff `ownership ∈ {owned, leased}`; null iff `ownership = sub_contracted` |
| `ownership` | enum | `owned` \| `leased` \| `sub_contracted` |
| `subcontractor_id` | string? | FK to [[Subcontractor]]. Required iff `ownership = sub_contracted`. |
| `status` | enum | `active` \| `in_maintenance` \| `out_of_service` \| `disposed` |
| `purchase_date` | date? | Owned only. Drives the depreciation rule per [[ADR-004]]. |
| `disposal_date` | date? | Set when `status` transitions to `disposed`. See [[ADR-018]]. |
| `current_odometer` | decimal? | Owned/leased only. Updated by latest WO close. |
| `notes` | string | |

**Constraints:**

- `plate_number IS NULL ⟺ ownership = sub_contracted`
- `subcontractor_id IS NOT NULL ⟺ ownership = sub_contracted`
- `purchase_date IS NOT NULL` if `ownership = owned`
- One sub-contract Vehicle per `(subcontractor_id, vehicle_type_id)` — a partner has one slot per type they offer.

## Ownership Modes

| Mode | What the row represents | Cost shape |
|---|---|---|
| `owned` | Physical truck. Has plate, odometer, depreciation rule per [[ADR-004]], CPK history per [[ADR-016]]. | Full per-WO formula: fixed + variable + driver + events |
| `leased` | Physical truck. Has plate, odometer; depreciation rule replaced by lease cost rule. CPK history accumulates. | Full per-WO formula |
| `sub_contracted` | Partner's supply slot. No plate, no odometer, no fixed cost rule, no CPK. | `subcontract_payout` Cost Event + optional incidentals |

The cost formula branches on `ownership` at WO cost-calculation time — see [[ADR-029]].

## Status Lifecycle

```
active         — operable, can be assigned to WOs
  ↓
in_maintenance — temporarily unavailable (scheduled service); returns to active
  ↓
out_of_service — long-term down (waiting for parts, accident); may return or proceed to disposed
  ↓
disposed       — terminal. Sale or write-off recorded. disposal_date set.
```

Sub-contract slots use `active` and a paused `out_of_service` only — `in_maintenance` and `disposed` are in-house concepts.

## Sub-Contracted Slots — Worked Example

A tenant has three partner companies:

| vehicle_id | ownership | subcontractor_id | vehicle_type | plate_number |
|---|---|---|---|---|
| V-100 | sub_contracted | Partner X | 22-wheel | null |
| V-101 | sub_contracted | Partner X | 10-wheel | null |
| V-102 | sub_contracted | Partner Y | 4-wheel | null |
| V-103 | sub_contracted | Partner Z | 22-wheel | null |

Four rows, total. Across hundreds of sub-contract trips, this stays four rows. Each trip's actual plate goes on `WorkOrder.subcontract_actual_plate`, never as a new Vehicle row.

VRP sees these four rows alongside owned/leased rows in the same supply pool. Cost evaluation per row reads [[Subcontractor Rate]] for sub-contracted; per-WO formula for in-house.

## Maintenance Linkage

Maintenance for owned/leased vehicles is recorded as [[Cost Event]]s with `entity_type = vehicle`, `entity_id = vehicle_id`. These feed the maintenance and tire CPK buckets per [[ADR-016]] and the per-WO `variable_cost` term in [[ADR-023]]. See [[Vehicle Maintenance — Overview]] and [[Integration — Vehicle Maintenance × Cost Ledger]].

Sub-contract slots have no maintenance — partner's responsibility.

## Depreciation and Disposal

Owned vehicle purchase creates a [[Cost Event]] (`cost_type = purchase`) and a depreciation [[Cost Rule]] per [[ADR-004]]. Disposal closes the rule and creates a (possibly negative) [[Cost Event]] per [[ADR-018]].

Leased vehicle has no purchase event; instead a `fixed_time` [[Cost Rule]] represents the lease payment.

Sub-contract slot has neither — no purchase, no depreciation, no disposal.

## Held for v2

| Field | Why deferred |
|---|---|
| `acquired_at` audit, owner contact | Multi-tenant ownership detail; only matters when the asset register expands beyond the trip system. |
| `gps_device_id`, telematics linkage | When live tracking lands. Currently odometer is captured at WO close. |

Capacity (`max_payload_kg`, `max_volume_m3`, `axle_count`) lives on [[Vehicle Type]] — same value for all instances of a type — not on Vehicle.

## Related

- [[Vehicle Type]] — class catalog this Vehicle belongs to
- [[Subcontractor]] — partner company (sub_contracted only)
- [[Subcontractor Rate]] — VRP cost estimate for sub-contracted
- [[Work Order]] — Vehicle is referenced via the [[Work Order Vehicle]] junction
- [[Work Order Vehicle]] — junction with role (tractor / trailer)
- [[Cost Event]] · [[Cost Rule]] · [[Cost Per Km]]
- [[Vehicle Maintenance — Overview]]
- [[Integration — Vehicle Maintenance × Cost Ledger]]
- [[Integration — Fuel Log × Cost Ledger]]
- [[ADR-004 — Vehicle Purchase Depreciation Spread]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-018 — Negative Amounts and Disposal]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
- [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]]
