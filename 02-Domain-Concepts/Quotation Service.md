# Quotation Service

#concept #module/order-mgmt #status/active

> Catalog entry. One row per `(quotation, service, vehicle_type)` triple, owning a [[Price Rule]] and an [[Allocation Rule]]. The menu the operator picks from at order time.

## Definition

Quotation Service is the price catalog entry. It answers: "For Customer X (via this Quotation), running Service Y in vehicle class Z, the cost is computed how?"

A Quotation contains N Quotation Services — the full menu of what can be ordered under that contract, broken out per supported vehicle type. The same Service appears once per supported vehicle type, each with its own pricing.

This is where [[Price Rule]] and [[Allocation Rule]] live at the catalog level. They are copied (snapshot) to [[Order Line]] at order time.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `quotation_service_id` | string | Unique identifier |
| `quotation_id` | string | Parent [[Quotation]] |
| `service_id` | string | Referenced [[Service]] (per-customer; the Service's customer must match the Quotation's customer) |
| `vehicle_type_id` | string | Vehicle class this price applies to (4-wheel, 10-wheel, 18-wheel, ...). Part of unique key. |
| `quoted_distance_km` | decimal | Lane distance used by `per_km` PriceRule. Set at catalog time. |
| `label` | string | Optional override display name for this customer's context |
| `status` | enum | `active` / `inactive` |

**Unique constraint:** `(quotation_id, service_id, vehicle_type_id)`. See [[ADR-021 — QuotationService Keyed by Vehicle Type]].

## Price Rule

Each Quotation Service owns one active [[Price Rule]] — the pricing method and params for *this customer × this service × this vehicle type*.

Close-and-replace when price is renegotiated. Existing [[Order Line]] snapshots are unaffected.

`per_km` reads `quoted_distance_km` from the OrderLine snapshot (copied from this Quotation Service), not from VRP route distance. See [[ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped]].

## Allocation Rule

Each Quotation Service owns a default [[Allocation Rule]] — how to split OrderLine revenue across its drop stops for ops-view analytics.

## What Gets Copied to Order Line

When operator creates an Order and selects this Quotation Service:

```
QuotationService.PriceRule         → copied → OrderLine.PriceRule (snapshot)
QuotationService.AllocationRule    → copied → OrderLine.AllocationRule (snapshot)
QuotationService.quoted_distance_km → copied → OrderLine.quoted_distance_km
QuotationService.service_id        → OrderLine.service_id (traceability)
QuotationService.vehicle_type_id   → OrderLine.vehicle_type_id (traceability + WO routing input)
Service.ServiceStops               → copied → OrderLine.OrderLocations (adjustable)
```

## Order-Time Selection Flow

1. Operator picks Customer → Quotation loads.
2. Operator picks a Service from this Quotation's catalog.
3. Operator picks a vehicle type from the supported set for that Service.
4. The matching `(quotation, service, vehicle_type)` row resolves.
5. Snapshots above are copied to a new OrderLine.

If only one vehicle type is offered for a Service, step 3 collapses to default selection.

## Related

- [[Quotation]] — parent
- [[Service]] — per-customer route template referenced
- [[Price Rule]] — owned by this entity
- [[Allocation Rule]] — owned by this entity
- [[Order Line]] — runtime instance
- [[ADR-007 — PriceRule Mirrors CostRule]]
- [[ADR-021 — QuotationService Keyed by Vehicle Type]]
- [[ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped]]
