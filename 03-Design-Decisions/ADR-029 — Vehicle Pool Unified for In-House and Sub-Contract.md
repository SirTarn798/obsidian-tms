# ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract

#decision #module/work-order #module/cost-ledger #status/active

## Status

Accepted. Establishes the [[Vehicle]] entity (previously implicit) and integrates sub-contracted vehicles into the same pool via supply slots. Adds [[Subcontractor]] and [[Subcontractor Rate]] as standing entities.

## Context

Two threads converge on this decision.

1. **No formal Vehicle concept doc.** [[Vehicle Type]] is documented; the Vehicle entity itself is referenced from [[Work Order]], [[Cost Ledger — Overview]], [[Fuel Log]], [[Integration — Vehicle Maintenance × Cost Ledger]], etc. but never defined. The implicit model — "instance of a Vehicle Type with an id and an odometer" — is too thin once ownership lifecycle, status, and maintenance linkage are real.

2. **Sub-contracted vehicles are a first-class execution mode.** Some tenants run partly or even entirely on partner-supplied trucks. Each trip pays a flat per-trip amount to the contractor; no fuel pool, no maintenance bucket, no driver salary. The trucks are also routinely ad-hoc — different plates trip after trip, often a different contractor company, sometimes a truck never seen again. The standing relationship is with the **partner company**, not the truck.

Three options were considered:

- **Flag on Vehicle (`ownership = sub_contracted`).** Single pool, but with ad-hoc trucks every trip would create a one-shot Vehicle row. Vehicle table bloats with throwaway rows; 90% of fields are null.
- **Separate `SubcontractedTrip` entity.** Eliminates branches in cost code, but VRP, fleet reports, vehicle-type filters, and licence/overlap checks all have to UNION two tables. Read paths pay back the savings many times over.
- **Unified pool with the realisation that the sub-contracted Vehicle row represents partner capacity for a vehicle type, not a physical truck.** A "Partner X — 22-wheel" Vehicle row is a long-lived supply slot, used across many WOs. The actual plate the partner sends is captured per-trip on the Work Order, not by creating new Vehicle rows.

The third reading makes the unified pool work without bloat.

## Decision

**The Vehicle entity holds both in-house and sub-contracted vehicles. A standing Subcontractor entity represents partner companies. A Subcontractor Rate carries per-trip cost rates for VRP estimation. The cost formula branches on `Vehicle.ownership` at WO cost-calculation time.**

### Vehicle Entity

| Field | Notes |
|---|---|
| `vehicle_id` | PK |
| `vehicle_type_id` | FK to [[Vehicle Type]] |
| `plate_number` | nullable. Required iff `ownership ∈ {owned, leased}`; null iff `ownership = sub_contracted` |
| `ownership` | enum: `owned` \| `leased` \| `sub_contracted` |
| `subcontractor_id` | nullable FK to [[Subcontractor]]. Required iff `ownership = sub_contracted` |
| `status` | enum: `active` \| `in_maintenance` \| `out_of_service` \| `disposed` |
| `purchase_date`, `disposal_date` | nullable; populated for owned/leased only. Tie to [[ADR-004]] / [[ADR-018]]. |
| `current_odometer` | nullable; tracked on owned/leased only |
| `notes` | |

**Constraints:**

- `plate_number IS NULL ⟺ ownership = sub_contracted`
- `subcontractor_id IS NOT NULL ⟺ ownership = sub_contracted`
- One sub-contracted Vehicle row per `(subcontractor_id, vehicle_type_id)` — a partner has one slot per type they offer.

In-house Vehicle = a specific physical truck. Sub-contracted Vehicle = a partner's standing offer to supply a vehicle of the given type, used across many trips.

### Subcontractor Entity

| Field | Notes |
|---|---|
| `subcontractor_id` | PK |
| `name`, `contact_name`, `contact_phone` | |
| `commercial_license_ref`, `insurance_ref`, `insurance_expiry` | optional; soft-block extension |
| `notes` | |

One Subcontractor → many Vehicle rows (one per `vehicle_type_id` they supply).

### Subcontractor Rate

Per-trip rate, mirroring [[Price Rule]] / [[Cost Rule]] structure. Used by VRP at planning time to estimate sub-contract cost. Replaced at WO close by the actual `subcontract_payout` [[Cost Event]].

| Field | Notes |
|---|---|
| `subcontractor_id`, `vehicle_type_id` | scope |
| `method`, `params` | `flat` / `per_km` / `per_stop` / `tiered` |
| `effective_from`, `effective_until` | close-and-replace per [[ADR-001]] |

### Work Order Cost Formula

Cost branches on the assigned Vehicle's ownership. No `execution_mode` field on Work Order — derived from the Vehicle.

```
ownership ∈ {owned, leased}:
  WO.cost = fixed_cost_share + variable_cost + driver_cost + events_cost
            (existing formula per [[ADR-023]])

ownership = sub_contracted:
  Pre-trip:   WO.cost ≈ SubcontractorRate.evaluate(...) + planned events_cost
  Post-trip:  WO.cost = subcontract_payout CostEvent + actual events_cost
              (fixed / variable / driver terms skipped)
```

The `subcontract_payout` is a [[Cost Event]] with `cost_classification = one_off`, `cost_type = subcontract_payout`, `work_order_id = wo.id`. Lands in the existing `events_cost` term — no new term introduced.

### Per-Trip Capture on Sub-Contract Work Order

When the partner confirms which truck and driver they sent, the controller fills:

- `WorkOrder.subcontract_actual_plate` — string, the partner's tractor / rig plate (one plate per the design discussion).
- On [[Work Order Driver]] (sub-contract case): `driver_id` is null; denormalised fields hold the contractor driver's details:
  - `external_driver_name`
  - `external_driver_phone`
  - `external_driver_license_class`
  - `external_driver_license_expiry`

Sub-contract drivers are not modelled as standing rows in [[Driver]] in v1 — they rotate freely and the cost of dedup logic outweighs the analytics gain.

### ADR Carve-Outs

| ADR | Carve-out |
|---|---|
| [[ADR-026 — Vehicle Type License Requirement and Driver Filter]] | Applies to both. Sub-contract case reads denormalised `external_driver_license_*` fields on [[Work Order Driver]]. |
| [[ADR-028 — Driver and Vehicle Overlap Soft-Block]] vehicle overlap | Skipped for sub-contract slots. Partner manages allocation of their capacity. |
| [[ADR-028]] driver overlap | In-house drivers: unchanged. External drivers: deferred — name+phone matching across WOs is fragile. |
| [[ADR-019 — Driver Salary Allocation by Duration Share]] | Skipped when `WorkOrderDriver.driver_id IS NULL`. |
| [[ADR-016 — Cost Per Km Buckets]], `fixed_time` Cost Rules | Naturally absent on sub-contract Vehicles (no events, no rules attached). No special-casing. |

### VRP Implication

Both pools are visible to VRP as a single supply pool. Each Vehicle row carries a cost model:

- In-house: full per-WO formula with CPK + fixed share + driver.
- Sub-contract: SubcontractorRate evaluation.

VRP optimises route + assignment as one problem with one pool. Replaces the previous-system hack where sub-contractor pricing was forced into Vehicle Type variants ("4-wheeler (X company)").

## Consequences

- **Good:** Single pool for VRP, fleet views, vehicle-type queries. No UNION of separate tables.
- **Good:** Sub-contracted Vehicle rows stay small (one per partner × type). No per-trip bloat.
- **Good:** [[Subcontractor Rate]] makes sub-contract cost comparable to in-house cost at planning time.
- **Good:** Cost formula has one branch, in one place (`vehicle.ownership` switch).
- **Good:** Adding a new partner is two writes — Subcontractor row + N Vehicle rows for supported types — plus one [[Subcontractor Rate]] per type.
- **Tradeoff:** `plate_number` is nullable; UI must handle the supply-slot case. Mitigated by the `ownership ⟺ plate_number` constraint and by `subcontract_actual_plate` on the WO.
- **Tradeoff:** Sub-contract drivers don't accumulate a profile — no usage history per individual. Acceptable; we trade per-driver analytics for not bloating the [[Driver]] table with one-shot rows.
- **Tradeoff:** [[Subcontractor Rate]] maintenance is operator-side discipline. Stale rates affect VRP comparison, not actual cost — the truth is the `subcontract_payout` CostEvent at WO close.

## Related

- [[Vehicle]]
- [[Subcontractor]]
- [[Subcontractor Rate]]
- [[Work Order Vehicle]]
- [[Vehicle Type]]
- [[Work Order]]
- [[Work Order Driver]]
- [[Cost Event]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]
- [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]]
