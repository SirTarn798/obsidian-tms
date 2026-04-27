# ADR-006 — Revenue Lives on Order, Not Work Order

#decision #module/order-mgmt #module/work-order #status/active

## Status

Accepted

## Context

VRP can merge stops from multiple [[Order]]s onto a single [[Work Order]], and can re-plan by splitting or merging Work Orders at any time. This creates a many-to-many relationship between Orders and Work Orders.

The original design stored income per [[Order Location]] inside `sub_cost` and summed it to get Work Order income. This was brittle: re-planning broke the sums, and income data was scattered across a table designed for cost data.

Meanwhile, invoicing is per-customer per-period, which maps naturally to Orders — not Work Orders.

## Decision

**Revenue is owned by Order. Work Order has display revenue only.**

### Canonical (Finance / Invoice)

- `order.revenue` — computed from [[Price Rule]], stored as snapshot at Order creation
- `order.billable_revenue` — what actually gets invoiced (may differ per [[ADR-011 — Partial Invoice Policy per Customer]])
- Invoice = `Σ order.billable_revenue WHERE customer=X AND period=P`
- Work Orders never appear on customer invoices

### Ops View (Display Only)

- `work_order.display_revenue` — computed on read from `Σ order_location.allocated_share` for done drop stops
- `work_order.display_margin` — `display_revenue − cost`
- Used for route profitability dashboards
- Clearly labeled as estimate/display in UI

### The Gap Is Information

Just as [[ADR-005 — Finance vs Operations Cost Views]] shows idle cost as the gap between finance and ops cost views, the gap between `Σ display_revenue across WOs` and `Σ order.revenue` reflects allocation methodology differences and incomplete routes. This is useful signal, not a bug.

## Consequences

- **Good:** Revenue is stable and correct regardless of VRP re-plans
- **Good:** Invoicing is simple — query Orders, not Work Orders
- **Good:** No income data in cost tables — clean separation
- **Tradeoff:** Work Order profitability is an estimate, not accounting truth
- **Tradeoff:** UI must clearly label `display_revenue` as ops-view, not invoice-view
