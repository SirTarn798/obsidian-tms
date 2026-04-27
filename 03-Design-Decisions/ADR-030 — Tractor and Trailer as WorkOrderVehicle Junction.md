# ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction

#decision #module/work-order #module/cost-ledger #status/active

## Status

Accepted. Replaces `WorkOrder.vehicle_id` (single FK) with a [[Work Order Vehicle]] junction. Mirrors the [[Work Order Driver]] shape established by [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]].

## Context

A 22-wheel rig is a tractor plus a trailer — two physical assets paired for one trip. Today [[Work Order]] holds a single `vehicle_id`. The realities behind the change:

- **Trailers swap.** Tractor T1 pulls trailer R1 today and trailer R2 tomorrow; R1 then goes with tractor T2. The pairing is per-trip, not fixed.
- **Maintenance is logged per asset.** Engine work goes against the tractor; brakes, axles, trailer body against the trailer. Tires can be either. CPK buckets need to attribute correctly per asset.
- **Multi-trailer cases exist.** B-doubles (1 tractor + 2 trailers) and similar configurations.
- **Bobtail repositioning.** A tractor sometimes runs alone (no trailer) to fetch a load.
- **[[ADR-028]] overlap** must catch trailer double-booking ("R1 booked on two tractors at the same time"), not just tractor double-booking.

A single `vehicle_id` collapses all of this. A second `trailer_id` field would be slightly cheaper to add but doesn't generalise to multi-trailer cases or to overlap symmetry.

## Decision

**Replace `WorkOrder.vehicle_id` with a [[Work Order Vehicle]] junction. One row per assigned asset on the WO, with `role` distinguishing tractor from trailer.**

### Schema

[[Work Order Vehicle]] junction:

| Field | Type | Description |
|---|---|---|
| `work_order_vehicle_id` | string | PK |
| `work_order_id` | string | FK |
| `vehicle_id` | string | FK |
| `role` | enum | `tractor` \| `trailer` |
| `assigned_at` | datetime | Audit — row creation |
| `added_at` | datetime | When the asset effectively joins the trip |
| `removed_at` | datetime? | NULL while asset is on the trip |
| `replaced_by_vehicle_id` | string? | If asset was swapped mid-trip, points at the replacement |
| `overlap_override_by_user_id` | string? | Per [[ADR-028]] |
| `overlap_override_at` | datetime? | |
| `overlap_override_reason` | string? | Free text. Required when override fires. |

**Constraints:**

- Exactly one `role = tractor` row per WO where `removed_at IS NULL`.
- Zero or more `role = trailer` rows.
- Unique `(work_order_id, vehicle_id)` — same asset cannot appear twice on one WO.
- Once `removed_at` is set, the row is immutable.

`role = tractor` semantically means **the powered, driver-occupied asset**. A 4-wheel solo truck has one row, role = tractor. A heavy tractor unit (the prime mover of a 22-wheel rig) is also role = tractor. The naming covers self-propelled vehicles regardless of whether they tow.

### WorkOrder.vehicle_id Removed

`WorkOrder.vehicle_id` is dropped. Reads that need "the tractor" pull `WorkOrderVehicle WHERE role = 'tractor'`.

WO odometer fields (`odometer_start`, `odometer_end`, `total_distance_actual_km`) stay on `WorkOrder` and document as the **tractor's** readings. Trailers usually have no odometer; if a future case needs per-trailer hub-meter readings, move to the junction then.

### Cost Formula

[[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]] sums per-component as before, with `fixed_cost_share` and `variable_cost` now generalised across assets:

```
WO.fixed_cost_share = Σ over WorkOrderVehicle rows where ownership ∈ {owned, leased}:
                        vehicle.daily_fixed_share

WO.variable_cost    = Σ over WorkOrderVehicle rows where ownership ∈ {owned, leased}:
                        Σ over applicable buckets:
                          cpk(vehicle, bucket, as_of=wo.date) × wo.distance_actual_km

  Tractor buckets: fuel + maintenance + tire
  Trailer buckets:        maintenance + tire   (no fuel)
```

For sub-contracted vehicles, see [[ADR-029]] — fixed/variable terms skipped.

`driver_cost`, `events_cost` unchanged. Per-stop split rules in [[ADR-023]] operate on the per-asset totals as before.

### Overlap Check

[[ADR-028 — Driver and Vehicle Overlap Soft-Block]] vehicle overlap generalises naturally — check overlap for each `(work_order, vehicle)` row. Catches trailer double-booking and tractor double-booking with the same query shape. Override fields move from `WorkOrder` to the junction row, mirroring the driver-overlap fields on [[Work Order Driver]].

### Licence Check

[[ADR-026 — Vehicle Type License Requirement and Driver Filter]] reads the **tractor's** `vehicle_type_id` to pick the licence requirement. Trailers do not drive the licence requirement. Query: `WorkOrderVehicle WHERE work_order_id = ? AND role = 'tractor'`.

### Mid-Trip Asset Swap

Mirrors helper-add per [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]. Stamp `removed_at` on the outgoing asset's row, insert a new row with the replacement, link via `replaced_by_vehicle_id`. Cost attribution reads time-bounded membership. Overlap check fires on the new row.

Trailer swap mid-trip follows this pattern in place. Tractor change mid-trip goes through full WO migration per [[ADR-010 — Work Order Migration on Failure]] — same path as today, since the tractor change implies the driver assignment frame also shifts.

### Sub-Contract Case

Per [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]], sub-contract WOs use one `WorkOrderVehicle` row, role = tractor, pointing at the partner's rig slot Vehicle. No trailer rows. The partner's internal pairing is opaque; we record one plate via `WorkOrder.subcontract_actual_plate`.

### VRP Implication

VRP gains a two-resource selection problem: tractor + trailer set per route. For v1, VRP returns a route + a tractor; the controller picks trailer(s) manually. Pairing-aware VRP is a future module change — flagged for the VRP overview when that module is designed.

## Consequences

- **Good:** Per-asset cost attribution. Trailer maintenance and tractor fuel attribute to the right asset's CPK.
- **Good:** Symmetric overlap check via the junction.
- **Good:** Generalises to multi-trailer (B-double) without schema change.
- **Good:** Bobtail repositioning fits naturally (1 tractor, 0 trailers).
- **Good:** Same shape as [[Work Order Driver]] — controllers and developers learn one junction pattern.
- **Tradeoff:** Schema migration from `WorkOrder.vehicle_id` to junction. One-shot.
- **Tradeoff:** UI complexity — assignment screen now has tractor field + trailer multi-select.
- **Tradeoff:** v1 VRP isn't pairing-aware. Controllers handle trailer selection. Acceptable until VRP gains the capability.

## Related

- [[Work Order Vehicle]]
- [[Work Order]]
- [[Work Order Driver]]
- [[Vehicle]]
- [[Vehicle Type]]
- [[Cost Per Km]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]
- [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
