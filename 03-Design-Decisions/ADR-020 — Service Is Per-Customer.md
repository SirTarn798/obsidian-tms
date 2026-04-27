# ADR-020 — Service Is Per-Customer

#decision #module/order-mgmt #status/active

## Status

Accepted. Replaces the "global reusable Service template, shared across customers" framing in earlier drafts of [[Service]] and [[Order Management — Overview]].

## Context

The original design framed [[Service]] as a globally shared route template. Multiple customers would reference the same Service; price came from each customer's [[Quotation Service]] entry.

In practice this never holds:

- Stops on a Service are real physical sites. A Service "BKK → CNX" for Customer A starts at A's warehouse and ends at A's distribution centre. Customer B's "BKK → CNX" starts and ends at B's sites. The two have nothing in common except a name.
- Service Stops reference [[Customer Location]]s, which belong to one customer. A "shared" Service with stops at one customer's locations is not actually shareable.
- Time windows, sequencing constraints, goods descriptions, and operational notes on a Service are customer-specific.
- The cross-customer reuse case ("popular routes between cities") is a reporting question, not a data-modelling one — it is answered by grouping on `service.name`, not by sharing rows.

Treating Service as global forced workarounds (per-customer overrides on Quotation Service, conditional Service Stops) and obscured the natural ownership.

## Decision

**Service is per-customer.** Every [[Service]] row carries `customer_id` and is owned by exactly one [[Customer]].

### Schema

| Field | Notes |
|---|---|
| `customer_id` | **NEW.** FK to Customer. Required. |
| `service_id`, `name`, `notes`, … | Existing fields kept. |

[[Service Stop]] rows on a Service reference the owning customer's [[Customer Location]]s only — enforceable as an integrity check.

### What this changes

- A Service "BKK → CNX" exists once per customer that runs that route. Same name across customers is normal and expected.
- [[Quotation Service]] still binds `(quotation, service, vehicle_type)` to a [[Price Rule]] and [[Allocation Rule]]. The Service-side ownership is now consistent: a Quotation for Customer A can only reference Services owned by A.
- Cross-customer reporting groups on `service.name`, not `service_id`. See the trace test in [[Model — Order to Work Order Chain]].

### What this does *not* change

- The Service → Service Stop → Customer Location chain.
- The catalog/pricing layer (still QuotationService).
- Order-time copy semantics ([[ADR-009 — OrderLocation as Snapshot of CustomerLocation]]).

## Consequences

- **Good:** Ownership of a Service is unambiguous — one customer, one set of allowed locations.
- **Good:** No more global vs per-customer duality on Service or Service Stop.
- **Good:** Naming collisions across customers are expected, not a data integrity problem.
- **Tradeoff:** Two customers running the *same* lane both maintain their own Service row. This is fine — the stops are different anyway.
- **Tradeoff:** Cross-customer "popular route" analytics must group by `service.name`, accepting that name normalisation is a reporting concern.

## Related

- [[Customer]]
- [[Service]]
- [[Service Stop]]
- [[Customer Location]]
- [[Quotation Service]]
- [[Model — Order to Work Order Chain]]
- [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]]
