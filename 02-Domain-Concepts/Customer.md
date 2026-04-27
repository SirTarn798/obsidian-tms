# Customer

#concept #module/order-mgmt #status/active

> The owning party for Quotations, Services, Customer Locations, and Orders. Top of the order-side ownership chain.

## Definition

A Customer is the commercial counter-party. Everything on the order side — [[Quotation]]s, [[Service]]s, [[Customer Location]]s, [[Order]]s — belongs to exactly one Customer. The platform never shares these entities across customers.

Customer holds the default invoice policy that propagates down through Quotation to Order; it is the resolver of last resort for the partial-invoice question ([[ADR-011 — Partial Invoice Policy per Customer]]).

## Key Fields

| Field | Type | Description |
|---|---|---|
| `customer_id` | string | Unique identifier |
| `name` | string | Display name |
| `legal_name` | string | Full legal entity name (invoicing) |
| `tax_id` | string | Tax registration |
| `contact_name` | string | Primary contact |
| `contact_phone` | string | |
| `contact_email` | string | |
| `billing_address` | string | Default invoice address |
| `default_invoice_policy` | enum | `all_or_nothing` \| `per_stop`. Default policy. |
| `status` | enum | `active` / `inactive` |
| `created_at` | datetime | |

## Ownership Chain

```
Customer
  ├── N Customer Location  (master sites)
  ├── N Service            (per-customer routes — see ADR-020)
  ├── N Quotation
  │     └── N Quotation Service  → Service + vehicle_type + PriceRule + AllocationRule
  └── N Order
        └── N Order Line  → Order Locations → Work Order Stops → Work Order
```

Each child entity carries `customer_id` directly or transitively. Cross-customer references are never created.

## Default Invoice Policy

`default_invoice_policy` defines whether an unfinished Order bills the full agreed amount or only the completed stops:

| Policy | Effect on `Order.billable_revenue` |
|---|---|
| `all_or_nothing` | Bill `Order.revenue` only if all eligible drop OrderLocations are `done`; else `0`. |
| `per_stop` | Bill `Σ allocated_share WHERE status=done AND stop_type=drop` regardless of completeness. |

Resolution order for a given Order:

```
1. Order.invoice_policy_override   (set explicitly on the Order)
2. Quotation.default_invoice_policy (inherited from contract)
3. Customer.default_invoice_policy  (this field — the floor)
```

See [[ADR-011 — Partial Invoice Policy per Customer]].

## Mutability

Customer fields are mutable master data. Changes do not propagate to historical records:

- `legal_name`, `tax_id`, `billing_address` updates apply only to invoices generated after the change.
- `default_invoice_policy` change applies only to Orders created after the change (existing Orders carry their own resolved policy).
- `name`, `contact_*` updates are reflected immediately in lookups; historical records render via snapshot data.

## Cross-Customer Reporting

Cross-customer analytics ("which lanes are busiest across all customers") group on `service.name` rather than `service_id` because Services are per-customer. See [[ADR-020 — Service Is Per-Customer]] and the trace test in [[Model — Order to Work Order Chain]].

## Related

- [[Quotation]] — commercial agreements
- [[Service]] — per-customer route catalog
- [[Customer Location]] — sites
- [[Order]] — placed orders
- [[Model — Order to Work Order Chain]]
- [[ADR-011 — Partial Invoice Policy per Customer]]
- [[ADR-020 — Service Is Per-Customer]]
