# Order

#concept #module/order-mgmt #status/active

> Customer commitment to run one or more services. The canonical unit for revenue attribution and invoicing. The stable record above the volatile [[Work Order]] layer.

## Definition

An Order represents what a customer wants done for a given trip or job. It contains one or more [[Order Line]]s — each an instance of a [[Quotation Service]] selected from the customer's [[Quotation]] catalog. Revenue is aggregated from Order Lines. Cost is aggregated from Order Lines (derived through OrderLocations and WorkOrderStops per [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]).

Execution is handled by [[Work Order]]s via [[Work Order Stop]]s. The Order layer never moves; VRP can re-shape WorkOrders freely below it.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `order_id` | string | Unique identifier |
| `customer_id` | string | [[Customer]] who placed the order |
| `quotation_id` | string | Source [[Quotation]] (traceability) |
| `status` | enum | `pending` → `in_progress` → `completed` \| `cancelled` |
| `closure_type` | enum | `auto` (all stops done) or `manual` (user forced close) |
| `closure_note` | string | Reason for manual close |
| `revenue` | decimal | `Σ OrderLine.revenue` — aggregated at confirmation |
| `cost` | decimal | **Derived.** `Σ OrderLine.cost`. Computed on read; cached at WO close. |
| `billable_revenue` | decimal | What to invoice — computed per policy, or user-confirmed on manual close |
| `billable_revenue_source` | enum | `computed` or `manual_override` |
| `effective_invoice_policy` | enum | `all_or_nothing` or `per_stop` — resolved from Order > Quotation > Customer |
| `invoice_policy_override` | enum | Optional explicit override for this Order |
| `created_at` | datetime | |
| `completed_at` | datetime | Set when `status` transitions to `completed`. Used by invoice query. |
| `cancelled_at` | datetime | Set when `status` transitions to `cancelled`. |

## Status Flow

```
pending      ─── operator confirms ──▶ in_progress
in_progress  ─── all stops terminal AND ≥1 done ──▶ completed
in_progress  ─── operator manually closes        ──▶ completed
pending|in_progress ─── operator cancels         ──▶ cancelled
```

`cancelled` requires no stops in `done`. If any stops are `done`, the Order must close as `completed` (with `closure_type=manual` and the unfulfilled portion handled per invoice policy).

## Structure

```
Order
  ├── OrderLine A  (e.g. "BKK → CNX" 10-wheel, revenue=15,000)
  │     ├── PriceRule snapshot
  │     ├── AllocationRule snapshot
  │     ├── quoted_distance_km
  │     └── OrderLocations: [OL1 pickup BKK, OL2 drop CNX]
  │
  └── OrderLine B  (e.g. "BKK → LPG" 4-wheel, revenue=12,000)
        ├── PriceRule snapshot
        ├── AllocationRule snapshot
        ├── quoted_distance_km
        └── OrderLocations: [OL3 pickup BKK, OL4 drop LPG]

Order.revenue = 15,000 + 12,000 = 27,000
Order.cost    = OrderLine A cost + OrderLine B cost  (derived through WorkOrderStops)
```

## Billable Revenue

`billable_revenue` = what appears on invoice.

- `all_or_nothing`: equals `revenue` only if all eligible drop OrderLocations across all OrderLines are `done`; else `0`.
- `per_stop`: `Σ allocated_share WHERE status=done AND stop_type=drop` across all OrderLines.

On manual close, operator confirms or overrides. `billable_revenue_source` tracks whether system-computed or user-set.

`effective_invoice_policy` resolution:
```
1. Order.invoice_policy_override   (if set)
2. Quotation.default_invoice_policy
3. Customer.default_invoice_policy
```

See [[ADR-011 — Partial Invoice Policy per Customer]].

## Cost (Derived)

`Order.cost = Σ OrderLine.cost`. Each OrderLine's cost rolls up from its OrderLocations, each of which sums across the WorkOrderStops referencing it (handles the migration case automatically). Computed on read per [[ADR-002 — Compute on Read, Cache When Needed]]; cached at WO close. See [[ADR-023]].

`Order.margin = Order.revenue − Order.cost` (display only). For invoiced margin, use `billable_revenue − cost`.

## Lifecycle

```
Services selected from Quotation catalog (per vehicle type)
  → Order created (status=pending)
  → OrderLines created, PriceRule/AllocationRule/quoted_distance_km snapshots copied
  → OrderLocations created from ServiceStops (adjustable until WO Stop)
  → Order confirmed: revenue computed, locations locked for VRP
  → VRP assigns stops to WorkOrders → status=in_progress
  → All eligible stops terminal OR user manually closes
  → status=completed (completed_at set) → included in next invoice run
```

Mid-flight amendments are append-only per [[ADR-024 — Quotation Amendment After In-Progress]].

## Invoice

Invoice = `Σ Order.billable_revenue WHERE customer=X AND completed_at IN period`.

[[Work Order]]s never appear on customer invoices.

## Related

- [[Customer]] — owner
- [[Order Line]] — service instances within this order
- [[Quotation]] — source catalog
- [[Work Order]] — execution (via Order Location → Work Order Stop)
- [[Model — Order to Work Order Chain]]
- [[Integration — Quotation × Order]]
- [[Integration — Order × Work Order]]
- [[Integration — Order × Cost Ledger]]
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-011 — Partial Invoice Policy per Customer]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-024 — Quotation Amendment After In-Progress]]
