# Order Line

#concept #module/order-mgmt #status/active

> One service instance within an [[Order]]. Owns a PriceRule snapshot, AllocationRule snapshot, quoted distance, stops, revenue, and (derived) cost.

## Definition

Order Line is the bridge between catalog (what customer can order) and execution (what gets done). Each Order Line represents one [[Service]] running under one vehicle type for an [[Order]]. A single Order can have multiple Order Lines — one per selected `(service, vehicle_type)` combination.

Order Line owns the price snapshot, the stop list (as [[Order Location]]s), and its computed revenue. The parent Order aggregates revenue and cost across all its Order Lines.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `order_line_id` | string | Unique identifier |
| `order_id` | string | Parent [[Order]] |
| `quotation_service_id` | string | Source [[Quotation Service]] (traceability) |
| `service_id` | string | Source [[Service]] (traceability and per-route reporting) |
| `vehicle_type_id` | string | Source vehicle type (traceability and WO routing input) |
| `quoted_distance_km` | decimal | Snapshot from QuotationService. Used by `per_km` / `tiered` PriceRule methods. |
| `revenue` | decimal | Computed from PriceRule snapshot — stored at confirmation |
| `cost` | decimal | **Derived.** `Σ OrderLocation.cost`. Computed on read; cached at WO close. |
| `price_manually_adjusted` | boolean | True if operator modified PriceRule params at order time |
| `status` | enum | Mirrors completion of its OrderLocations — see rule below |

## Price Rule

Copied from [[Quotation Service]] at order creation. Operator can adjust params (e.g. change flat amount) before confirming the order. Once confirmed, locked.

`price_manually_adjusted = true` when params differ from the Quotation Service source — flagged for audit/review.

## Allocation Rule

Copied from [[Quotation Service]] at order creation. Determines `allocated_share` per drop [[Order Location]] under this line.

## Order Locations

Created by copying [[Service Stop]]s from the source [[Service]]. Operator can:
- Add stops
- Remove stops
- Swap locations (different [[Customer Location]] of the same customer)
- Reorder sequence

All adjustments allowed before VRP runs. After a [[Work Order Stop]] is created for an OrderLocation, that location is **locked**. New OrderLocations may still be **appended** to the OrderLine even after lock; existing locked locations cannot be edited or reordered. See [[ADR-024 — Quotation Amendment After In-Progress]].

## Status — Mirroring Rule

`OrderLine.status` derives from its OrderLocations:

| Condition (over the line's drop OrderLocations) | OrderLine.status |
|---|---|
| Any in `planned` or `in_progress` (i.e. non-terminal) | `in_progress` |
| All terminal AND ≥1 in `done` | `completed` |
| All terminal AND none in `done` | `failed` |
| All in `cancelled` | `cancelled` |

Pickups are excluded from status mirroring — drops define delivery completion. (Pickups still count for cost.)

## Revenue Contribution

```
OrderLine.revenue = computed from PriceRule (flat / per_km × quoted_distance_km / per_weight / etc.)
Order.revenue     = Σ OrderLine.revenue
```

## Cost (Derived)

```
OrderLine.cost = Σ OrderLocation.cost
              -- where OrderLocation.cost rolls up from its WorkOrderStops
```

Per [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]. `OrderLine.margin = revenue − cost` (display).

## Billable Revenue

Order Line does not independently compute billable revenue. The parent [[Order]]'s `effective_invoice_policy` applies across all Order Lines together:

- `all_or_nothing`: bill all OrderLines' revenue only if all drop OrderLocations are done
- `per_stop`: bill `Σ allocated_share` of done drop OrderLocations across all OrderLines

## Traceability Chain

```
Order Line → Quotation Service → Quotation (contract)
          → Service (per-customer route template)
          → Vehicle Type
          → Order Locations → Work Order Stops → Work Order
```

## Related

- [[Order]] — parent
- [[Quotation Service]] — source of PriceRule, AllocationRule, quoted_distance_km
- [[Service]] — route template (per-customer)
- [[Order Location]] — the stops under this line
- [[Price Rule]]
- [[Allocation Rule]]
- [[Work Order Stop]] — execution join
- [[Model — Order to Work Order Chain]]
- [[ADR-021 — QuotationService Keyed by Vehicle Type]]
- [[ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-024 — Quotation Amendment After In-Progress]]
