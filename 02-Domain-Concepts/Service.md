# Service

#concept #module/order-mgmt #status/active

> Per-customer named route. Defines stop sequence only — no price. Price lives on [[Quotation Service]].

## Definition

A Service is one customer's named route — e.g. Customer A's "BKK → CNX" — described as an ordered set of template stops at that customer's [[Customer Location]]s. Services are **per-customer** ([[ADR-020 — Service Is Per-Customer]]); same name across different customers is normal and expected, but each customer maintains their own Service rows because the actual stops differ.

Service owns the *what* (which locations, in what order). [[Quotation Service]] owns the *how much* (PriceRule per vehicle type for that customer).

## Key Fields

| Field | Type | Description |
|---|---|---|
| `service_id` | string | Unique identifier |
| `customer_id` | string | Owning customer ([[Customer]]). Required. |
| `name` | string | Display name (e.g. "BKK → CNX", "ท่าเรือ → คลังสินค้า A") |
| `description` | string | Operational notes |
| `status` | enum | `active` / `inactive` |

## Service Stops

Each Service has N [[Service Stop]]s — the template stop sequence. Each Service Stop references a [[Customer Location]] **belonging to the same customer**.

```
Service "BKK → CNX"  (customer_id = CUS-A)
  ServiceStop 1: pickup, CustomerLocation "CUS-A คลังสินค้า BKK"
  ServiceStop 2: drop,   CustomerLocation "CUS-A คลังสินค้า CNX"
```

Stops are templates. When an [[Order Line]] is created from this Service, stops are copied as [[Order Location]]s and can be adjusted before VRP runs.

## Pricing — Per Vehicle Type

A single Service can be priced multiple ways in a Quotation, one per supported vehicle type. The composite key on [[Quotation Service]] is `(quotation_id, service_id, vehicle_type_id)`. See [[ADR-021 — QuotationService Keyed by Vehicle Type]].

```
Service "BKK → CNX"  (Customer A)
  ← QuotationService (vehicle_type=4-wheel):  PriceRule flat 12,000
  ← QuotationService (vehicle_type=10-wheel): PriceRule flat 15,000
  ← QuotationService (vehicle_type=18-wheel): PriceRule per_km 22 × quoted_distance_km
```

## Cross-Customer Reuse

Services are not shared across customers. If Customers A and B both run a "BKK → CNX" route, each maintains their own Service with their own Customer Locations. Cross-customer reporting groups on `service.name` (cosmetic) rather than `service_id` (would never match).

## Versioning

Service stops can be updated (add/remove/reorder locations). Existing [[Order Location]]s are snapshots — unaffected by Service changes (see [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]]).

## Related

- [[Customer]] — owner
- [[Service Stop]] — template stops within this Service
- [[Customer Location]] — where stops are
- [[Quotation Service]] — per-customer per-vehicle-type price link
- [[Order Line]] — order-time instance of this Service
- [[Driver Rate]] — service-scope rates attach here ([[ADR-025]])
- [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]]
- [[ADR-020 — Service Is Per-Customer]]
- [[ADR-021 — QuotationService Keyed by Vehicle Type]]
