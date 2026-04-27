# Order Management — Overview

#module/order-mgmt #status/active

> Captures customer intent and owns revenue. The stable layer VRP and Work Orders operate against.

## Context

Order Management answers: "What did the customer ask for, and what did we agree to charge?" It is the canonical source for revenue and invoicing. Work Order execution may change freely (VRP re-plans, vehicle swaps, partial failures) — the Order layer never moves.

The original design built VRP first, causing order and pricing data to be designed around route execution. This redesign reverses that: **Work Orders serve Orders, not the other way around.**

## How It Works

### Layer 1 — Master Data (per Customer)

[[Customer]] — the commercial counter-party. Owns Quotations, Services, Customer Locations, Orders. Holds default invoice policy.

[[Customer Location]] — reusable physical locations (customer warehouses, depots, terminals). Mutable master; changes do not affect existing orders.

[[Service]] — **per-customer** named route. Defines stop sequence only. See [[ADR-020 — Service Is Per-Customer]].

[[Service Stop]] — template stops within a Service. Reference the owning customer's Customer Locations. Copied to [[Order Location]]s at order time.

### Layer 2 — Service Catalog (Quotation)

[[Quotation]] — commercial contract per customer. Contains N [[Quotation Service]]s.

[[Quotation Service]] — keyed by `(quotation, service, vehicle_type)` per [[ADR-021 — QuotationService Keyed by Vehicle Type]]. Owns one [[Price Rule]] and one [[Allocation Rule]]. Same Service appears once per supported vehicle type at its own price. This is the order catalog the operator picks from.

### Layer 3 — Order Creation

When a customer places an order:

1. Operator selects Customer → Quotation loads
2. Operator picks one or more `(service, vehicle_type)` combinations from the catalog
3. For each selection → [[Order Line]] created:
   - [[Price Rule]] copied from QuotationService → OrderLine snapshot
   - [[Allocation Rule]] copied → OrderLine snapshot
   - `quoted_distance_km` copied → OrderLine
   - [[Service Stop]]s copied → [[Order Location]]s (address/geo snapshot from master)
   - `OrderLine.revenue` computed from Price Rule (using `quoted_distance_km` for `per_km` / `tiered`)
4. `Order.revenue = Σ OrderLine.revenue`
5. `OrderLocation.allocated_share` computed (drops only, pickups = 0)
6. Operator may adjust: price params, stop list, location overrides — before confirming
7. Order confirmed → locked for VRP. Mid-flight amendments append-only per [[ADR-024 — Quotation Amendment After In-Progress]].

### Layer 4 — Revenue Computation

`order.revenue` = aggregated from Order Line revenues. Each OrderLine revenue derived from its copied Price Rule method (flat, per_km × `quoted_distance_km`, per_weight, etc.).

`order.billable_revenue` = what gets invoiced. Computed per `effective_invoice_policy` as stops complete. Overridable on manual close.

### Layer 5 — Invoice

Invoice = `Σ order.billable_revenue WHERE customer=X AND completed_at IN period`.

Work Orders do not appear on invoices. Revenue flows: QuotationService (catalog price) → OrderLine (committed instance) → Order (aggregated) → Invoice (billed).

### Layer 6 — Cost Back to Order

Cost flows in the opposite direction through the same chain. Per [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]:

```
WorkOrderStop.cost  = variable + fixed_share + driver_share + events_share
OrderLocation.cost  = Σ wos.cost referencing this OrderLocation
OrderLine.cost      = Σ OrderLocation.cost
Order.cost          = Σ OrderLine.cost
```

Per-OrderLine and per-Order P&L are well-defined regardless of how VRP merged or split the executing WorkOrders.

## Key Design Principles

1. **Orders are the stable record.** VRP re-plans and WO changes never touch Order data.
2. **Revenue never depends on cost.** Price Rule reads order parameters (`quoted_distance_km`, weight), not Cost Ledger or VRP route distance.
3. **Snapshot on copy.** Quotation price changes don't corrupt existing Orders. OrderLine holds its own PriceRule, AllocationRule, and `quoted_distance_km` snapshot.
4. **Pickup stops earn 0 revenue.** Allocation distributes to drops only. Cost still flows through pickups.
5. **Service = template, QuotationService = price (per vehicle type), OrderLine = instance.** Three distinct concepts, not one.
6. **Service is per-customer.** Same name across customers is normal; rows are separate.
7. **Mid-flight amendments are append-only.** Locked OrderLines accept new OrderLocations but no edits to existing ones.

## Key Decisions

- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-007 — PriceRule Mirrors CostRule]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]]
- [[ADR-011 — Partial Invoice Policy per Customer]]
- [[ADR-020 — Service Is Per-Customer]]
- [[ADR-021 — QuotationService Keyed by Vehicle Type]]
- [[ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-024 — Quotation Amendment After In-Progress]]

## Scenarios

See [[Scenarios — Order to Invoice]] for end-to-end walkthroughs: happy path, VRP merge, re-plan, partial failure + migrate, per-stop billing, manual close.

## Connects To

- **[[Model — Order to Work Order Chain]]** — the central data model spanning Order, WorkOrder, and Cost Ledger. Read first.
- [[Integration — Quotation × Order]] — how quotation becomes order
- [[Integration — Order × Work Order]] — how stops flow to execution
- [[Integration — Order × Cost Ledger]] — delivery cost vs order revenue for profitability
- [[Work Order — Overview]] — execution layer

## Open Questions

- Should Quotation support line-item pricing per Customer Location (enabling `by_quoted_sub_price` allocation)?
- Multi-currency support — is it needed now or future?
