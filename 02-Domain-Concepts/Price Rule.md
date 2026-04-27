---
tags: [concept, module/order-mgmt, status/active]
---

# Price Rule

#concept #module/order-mgmt #status/active

> How revenue is computed for a scope. Mirrors [[Cost Rule]] in structure — time-ranged, close-and-replace, never mutated.

## Definition

A Price Rule defines the method and parameters for computing [[Order Line]] revenue. Attached to a [[Quotation Service]] and **copied** as a snapshot to each [[Order Line]] at order creation. The OrderLine's copy is immutable — changes to the QuotationService's Price Rule do not affect existing Orders.

Revenue never reads from cost. Revenue is also independent of VRP — `per_km` reads `quoted_distance_km` from the OrderLine, not from any Route or WorkOrderStop distance. See [[ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped]].

## Key Fields

| Field | Type | Description |
|---|---|---|
| `price_rule_id` | string | |
| `scope_type` | enum | `quotation_service` (catalog rule) or `order_line` (snapshot) |
| `scope_id` | string | ID of owning QuotationService or OrderLine |
| `method` | enum | See methods below |
| `params` | JSON | Method-specific parameters |
| `effective_from` | date | |
| `effective_until` | date | NULL = still active. (Naming aligned with [[Cost Rule]].) |

## Methods

| Method | Params | Revenue formula |
|---|---|---|
| `flat` | `{ amount }` | `amount` |
| `per_km` | `{ rate_per_km, min?, max? }` | `rate_per_km × OrderLine.quoted_distance_km` |
| `per_weight` | `{ rate_per_kg, min?, max? }` | `rate_per_kg × Σ order_location.weight_kg` |
| `per_stop` | `{ rate_per_stop }` | `rate_per_stop × count(drop stops)` |
| `tiered` | `{ tiers: [{up_to_km, rate}, ...] }` | Lookup by `OrderLine.quoted_distance_km` band |

`cost_plus` was removed by [[ADR-022]]. Bespoke cost-plus contracts use the manual-override path on `Order.billable_revenue` at close time.

## Quoted Distance

`per_km` and `tiered` read `OrderLine.quoted_distance_km`, copied from the [[Quotation Service]] at order time. This decouples revenue from VRP route distance and from per-WorkOrderStop leg distances. The customer is quoted on the agreed lane distance and that figure is locked at order confirmation.

## Close and Replace

To change a QuotationService's pricing: close the current rule (`effective_until = today`), create a new rule. Existing Orders with copied snapshots are unaffected.

Same principle as [[ADR-001 — Close and Replace, Never Mutate]].

See [[ADR-007 — PriceRule Mirrors CostRule]].

## Related

- [[Quotation Service]] — primary scope (catalog)
- [[Order Line]] — holds copied snapshot
- [[Allocation Rule]] — distributes computed revenue to stops
- [[Cost Rule]] — structural analog
- [[ADR-007 — PriceRule Mirrors CostRule]]
- [[ADR-021 — QuotationService Keyed by Vehicle Type]]
- [[ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped]]
