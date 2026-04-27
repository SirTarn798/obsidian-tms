# ADR-007 — PriceRule Mirrors CostRule

#decision #module/order-mgmt #status/active

## Status

Accepted. `cost_plus` method removed by [[ADR-022 — PriceRule per_km Uses Quoted Distance, cost_plus Dropped]].

## Context

Early design stored price as a flat scalar on Order. This prevented implementing other pricing methods (per km, per weight, tiered, cost-plus) without schema changes. The [[Cost Ledger]] already solved this pattern well for costs via [[Cost Rule]] — time-ranged, method-parameterised, close-and-replace, never mutated.

## Decision

**[[Price Rule]] mirrors [[Cost Rule]] in structure.**

| Property | Cost Rule | Price Rule |
|---|---|---|
| Scope | Vehicle / Driver / User (per [[ADR-013]]) | QuotationService (catalog) / OrderLine (snapshot) |
| Method | fixed / per_km | flat / per_km / per_weight / per_stop / tiered |
| Params | JSON | JSON |
| Time-ranged | Yes | Yes |
| Mutation policy | Close and replace | Close and replace |
| Snapshot on use | CostEvent references rule | OrderLine copies rule at order creation |

### Why copy to Order (not FK to Quotation rule)

If Order kept only a FK to the Quotation's Price Rule, and that rule is later closed and replaced, historical revenue computations would need to re-derive from the old rule. Copying the snapshot at Order creation makes the Order self-contained — revenue is always reproducible from the Order record alone.

### Revenue never depends on cost

No PriceRule method reads cost data. Earlier drafts included a `cost_plus` method; it was removed by [[ADR-022]] for breaking "revenue knowable at order confirmation." Bespoke cost-plus contracts use the manual-override path on `Order.billable_revenue`. There is no implicit loop where "location with more cost gets more revenue." That pattern was a workaround and is eliminated by this design.

## Consequences

- **Good:** New pricing methods addable without schema change — add a method + params
- **Good:** Order revenue always reproducible from the Order record alone
- **Good:** Quotation price changes never corrupt existing Orders
- **Tradeoff:** Slight data duplication (copied rule on each Order)
- **Tradeoff:** Changing a pricing method mid-quotation requires explicit close-and-replace discipline
