# ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped

#decision #module/order-mgmt #status/active

## Status

Accepted. Refines [[Price Rule]] method semantics defined in [[ADR-007 — PriceRule Mirrors CostRule]].

## Context

[[Price Rule]] supports several pricing methods. Two of them caused trouble:

### `per_km` reading VRP route distance

If `per_km` revenue reads `Route.distance_planned_km` or `Σ WorkOrderStop.distance_actual_km`, then revenue moves whenever VRP re-plans or the driver detours. That breaks the central design rule that the Order layer is stable and revenue is locked at order confirmation. It also means the customer cannot be quoted a firm price up-front — every re-route changes their bill.

The customer-facing distance is the *quoted* lane distance, not the *executed* trip distance. They are different numbers and they should not bleed into each other.

### `cost_plus`

`cost_plus` revenue = cost × (1 + margin_pct). It reads cost data into revenue computation. This:

- Couples revenue to operational cost — a cost recompute mutates historical revenue.
- Cannot produce a firm price at order confirmation — cost is not known until the WO closes.
- Cannot be invoiced cleanly — what value goes on the invoice when the customer asks for one immediately after delivery? An estimate? A live re-derivation?
- Is rare in practice — the few real cost-plus contracts are negotiated bespoke and reconciled manually.

The trace test in [[Model — Order to Work Order Chain]] requires revenue to be VRP-stable and knowable at order time. `cost_plus` fails both.

## Decision

### `per_km` reads `quoted_distance_km` from the OrderLine snapshot

```
OrderLine.revenue (per_km) =
    PriceRule.params.rate_per_km
  × OrderLine.quoted_distance_km
```

`OrderLine.quoted_distance_km` is set at order creation, copied from the [[Quotation Service]]'s PriceRule params (or as an explicit lane distance on the QuotationService). It does not change after order confirmation.

VRP route distance, actual trip distance, and per-WorkOrderStop leg distance are **operational** numbers used for cost computation. They never feed revenue.

### `cost_plus` is removed from PriceRule

| Method | Status |
|---|---|
| `flat` | Keep |
| `per_km` | Keep — uses `quoted_distance_km` |
| `per_weight` | Keep |
| `per_stop` | Keep |
| `tiered` | Keep |
| `cost_plus` | **Dropped** |

Existing references to `cost_plus` in concept docs and ADRs are updated to remove it.

### Bespoke cost-plus deals

For the rare contract that genuinely is "actual cost plus margin," the path is:

1. Use a regular method (commonly `flat`) at the quoted estimate.
2. At Order close, the operator manually overrides `Order.billable_revenue` with the actual figure.
3. The override is recorded on the Order, not derived from cost on the fly.

This keeps cost out of the revenue derivation path while still supporting the workflow.

## Consequences

- **Good:** Revenue is fully decoupled from VRP and from cost. The trace test holds in both directions.
- **Good:** Order revenue is firm at confirmation. No surprise re-computation post-execution.
- **Good:** PriceRule methods are a clean enum of customer-facing pricing models.
- **Tradeoff:** `quoted_distance_km` is a separate field from any operational distance — operators must understand the distinction.
- **Tradeoff:** Cost-plus contracts use the manual-override path instead of an automatic method. Acceptable given how rare they are.

## Related

- [[Price Rule]]
- [[Order Line]]
- [[Quotation Service]]
- [[Work Order Stop]]
- [[Route]]
- [[Model — Order to Work Order Chain]]
- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-007 — PriceRule Mirrors CostRule]]
- [[ADR-011 — Partial Invoice Policy per Customer]]
