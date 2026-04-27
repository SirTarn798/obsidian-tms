# Order Location

#concept #module/order-mgmt #status/active

> A single stop (pickup or drop) within an [[Order Line]]. Snapshot of where the vehicle goes. Locked once a [[Work Order Stop]] points to it.

## Definition

Order Location is the atomic unit of route execution on the Order side. Copied from a [[Service Stop]] at order time, then adjustable by the operator before VRP creates a [[Work Order Stop]] for it. Once a Work Order Stop exists, it is locked.

Captures customer intent as a snapshot — address and geo copied from [[Customer Location]] master at order time. Changes to the master do not affect existing Order Locations.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `order_location_id` | string | Unique identifier |
| `order_line_id` | string | Parent [[Order Line]] |
| `order_id` | string | Denormalised from OrderLine — for convenient querying |
| `customer_location_id` | string | Source master location (FK — reference only, fields are snapshot) |
| `stop_type` | enum | `pickup` or `drop` |
| `sequence` | integer | Intended order within this OrderLine's stops |
| `address` | string | Snapshot of address at order time |
| `lat` | decimal | Snapshot |
| `lng` | decimal | Snapshot |
| `time_window_start` | datetime | Requested arrival window |
| `time_window_end` | datetime | |
| `goods_description` | string | What is being moved at this stop |
| `weight_kg` | decimal | Used by weight-based [[Allocation Rule]] |
| `status` | enum | See status flow below |
| `allocated_share` | decimal | Revenue share from [[Allocation Rule]] — always 0 for pickups |
| `cost` | decimal | **Derived.** `Σ wos.cost WHERE wos.order_location_id = this.id`. Computed on read; cached at WO close. |

## Snapshot vs Master

`customer_location_id` references master for lookup. All address/geo fields are copied at order creation. Master updates do not affect existing Order Locations.

See [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]].

## Adjustability Window

| Phase | Can adjust? |
|---|---|
| Order created, not yet confirmed | Yes — add/remove/swap/reorder freely |
| Order confirmed, no WO Stop yet | Yes — until WO Stop created |
| WO Stop created (any status) | **Locked** — fields, sequence, location reference all immutable |
| `status=done` | Permanent record |

The OrderLine itself remains append-only on locked lines per [[ADR-024 — Quotation Amendment After In-Progress]] — new OrderLocations may be added even when existing ones are locked.

## Revenue Allocation

Only `drop` stops receive revenue allocation. Pickup stops always have `allocated_share = 0`.

`allocated_share` computed by parent [[Order Line]]'s [[Allocation Rule]], stored on each drop OrderLocation. Revenue is split across drop stops of the same OrderLine only — never cross-line.

## Cost (Derived)

```
OrderLocation.cost = Σ wos.cost
                     WHERE wos.order_location_id = this.id
```

Usually one WorkOrderStop per OrderLocation; multiple rows occur after migration ([[ADR-010 — Work Order Migration on Failure]]) when an old `migrated` stop and a new `done` stop both point at this OrderLocation. Both contribute. Per [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]].

Pickups have non-zero cost (they consume road) even though they have zero revenue allocation.

## Status Flow

```
planned  →  done       (driver completes stop)
planned  →  failed     (driver could not complete)
planned  →  cancelled  (customer/ops cancels)
planned  →  migrated   (transferred to new WorkOrder via Migrate)
```

Status mirrors up to OrderLine and Order per the rules in [[Order Line]] and [[Order]].

## Related

- [[Order Line]] — parent
- [[Order]] — grandparent (denormalised via `order_id`)
- [[Service Stop]] — template this was copied from
- [[Customer Location]] — master source
- [[Work Order Stop]] — join to WorkOrder
- [[Allocation Rule]] — determines `allocated_share`
- [[Model — Order to Work Order Chain]]
- [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-024 — Quotation Amendment After In-Progress]]
