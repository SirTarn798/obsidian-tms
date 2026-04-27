# Work Order Vehicle

#concept #module/work-order #module/cost-ledger #status/active

> Junction between [[Work Order]] and [[Vehicle]] that carries role (tractor / trailer) and per-asset time bounds. Replaces the single `vehicle_id` field on Work Order.

## Definition

A Work Order has **one tractor** and **zero or more trailers**. The Work Order Vehicle row captures each (Work Order, Vehicle) pair with its role and the time window during which that asset was on the trip. Same structural shape as [[Work Order Driver]].

Locked by [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]]. Replaces the `WorkOrder.vehicle_id` single-FK design.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `work_order_vehicle_id` | string | PK |
| `work_order_id` | string | Linked [[Work Order]] |
| `vehicle_id` | string | Linked [[Vehicle]] |
| `role` | enum | `tractor` / `trailer` |
| `assigned_at` | datetime | Audit — when the row was created |
| `added_at` | datetime | When the asset effectively joins the trip. Equals `assigned_at` for rows added during planning; `now()` for mid-trip swap-ins. |
| `removed_at` | datetime? | NULL while asset is on the trip. Set on swap-out. |
| `replaced_by_vehicle_id` | string? | If asset was swapped, points at the replacement |
| `overlap_override_by_user_id` | string? | Set when assignment overrode the vehicle-overlap soft-block. See [[ADR-028]]. |
| `overlap_override_at` | datetime? | |
| `overlap_override_reason` | string? | Free text. Required when override fires. |

**Unique constraint:** `(work_order_id, vehicle_id)` — same Vehicle cannot appear twice on one WO. Once `removed_at` is set, the row is immutable.

## Constraints

- Exactly one row per Work Order has `role = tractor` and `removed_at IS NULL`.
- Trailers are unbounded — zero or more.
- Bobtail (tractor running solo for repositioning) is one tractor row, zero trailer rows.
- Rows can be inserted/deleted while `WorkOrder.status = planned`.
- Once `WorkOrder.status` advances to `in_progress`:
  - Tractor rows are **frozen**. Tractor change goes through migration ([[ADR-010]]).
  - Trailer rows remain **mutable** — add/remove freely. Removal sets `removed_at`; the row stays (mirrors [[ADR-027]] for helpers).
- All rows immutable once `WorkOrder.status = completed | closed_partial | cancelled`.

## Role Semantics

`tractor` is **the powered, driver-occupied asset**. This covers:

- Heavy tractor units (the prime mover of a 22-wheel rig).
- Self-propelled trucks of any size that aren't tractors in the heavy-haulage sense (4-wheel pickups, 6-wheel rigid trucks, 10-wheel rigid trucks running solo).

The naming reflects "the asset that has the engine and the driver" rather than the heavy-haulage sense alone. A 4-wheel solo truck has one row, role = tractor.

`trailer` is a towed asset with no engine. No fuel CPK; tire and maintenance CPK only.

## Soft-Block Check at Assignment

Per [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]:

```
conflicts = WOs whose effective window overlaps this WO's window
            AND that have any non-cancelled WorkOrderVehicle row for this vehicle
            (excluding this WO and excluding rows where removed_at <= wo.window_start)

if conflicts non-empty:
  show conflict list
  require "Proceed anyway" + non-empty reason
  set overlap_override_* fields
```

Sub-contract slots are excluded from the overlap check per [[ADR-029]] — the partner manages allocation of their capacity.

## Cost Implication

Per [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]:

```
WO.fixed_cost_share = Σ over WorkOrderVehicle rows where ownership ∈ {owned, leased}:
                        vehicle.daily_fixed_share

WO.variable_cost    = Σ over WorkOrderVehicle rows where ownership ∈ {owned, leased}:
                        Σ over applicable buckets:
                          cpk(vehicle, bucket, as_of=wo.date) × wo.distance_actual_km

  Tractor buckets: fuel + maintenance + tire
  Trailer buckets:        maintenance + tire   (no fuel)
```

For sub-contracted vehicles, see [[ADR-029]] — fixed/variable terms skipped; WO cost is dominated by the `subcontract_payout` [[Cost Event]].

`driver_cost`, `events_cost` are unchanged. Per-stop split rules from [[ADR-023]] operate on the per-asset totals.

## Lifecycle

```
WO created (status=planned)
  → INSERT WorkOrderVehicle(wo, tractor_id, role=tractor, assigned_at, added_at = same)
    → overlap soft-block fires
  → optionally INSERT trailers
    → overlap soft-block fires per trailer

WO planning edits
  → INSERT/DELETE trailers freely
  → tractor may be swapped via DELETE+INSERT

WO starts (status=in_progress)
  → tractor row frozen
  → trailers still mutable:
      add trailer:    INSERT row with added_at = now() (overlap check fires)
      release trailer: UPDATE existing row, removed_at = now()
      swap trailer:    UPDATE outgoing removed_at + INSERT new row, link replaced_by_vehicle_id

Mid-trip tractor change → migration (ADR-010), not in-place

WO completed
  → all rows immutable
  → cost computed using junction rows + Vehicle.ownership branch
```

## Sub-Contract Case

Per [[ADR-029]] a sub-contract WO has exactly one `WorkOrderVehicle` row, `role = tractor`, pointing at the partner's rig slot Vehicle. No trailer rows. The partner's internal pairing is opaque; the actual rig plate is recorded on `WorkOrder.subcontract_actual_plate`.

## Related

- [[Work Order]]
- [[Vehicle]]
- [[Vehicle Type]]
- [[Work Order Driver]] — structural analog (driver junction)
- [[Cost Per Km]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]
- [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
- [[ADR-030 — Tractor and Trailer as WorkOrderVehicle Junction]]
