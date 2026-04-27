# ADR-021 — QuotationService Keyed by Vehicle Type

#decision #module/order-mgmt #status/active

## Status

Accepted. Refines [[Quotation Service]] composite key.

## Context

For a given customer Service, price typically depends on vehicle class — a 4-wheel run and a 10-wheel run on the same lane carry different rates because cost structures differ. The original [[Quotation Service]] PK was `(quotation_id, service_id)` — one row per Service per Quotation, single price.

Workarounds appeared:

- Duplicate Services per vehicle class (`BKK→CNX 4-wheel`, `BKK→CNX 10-wheel`) — pollutes the Service catalog with non-route data and breaks Service identity.
- A `vehicle_type` column on [[Price Rule]] params with method-specific handling — pushes vehicle-type pricing into method internals where it does not belong.
- One Quotation per vehicle class — explodes Quotation count and breaks the "one customer commercial agreement" framing.

The clean fix is to put vehicle type into the QuotationService key itself. Vehicle type is a first-class price dimension, not a method param.

## Decision

**[[Quotation Service]] is uniquely keyed by `(quotation_id, service_id, vehicle_type_id)`.** Same Service in the same Quotation may appear once per supported vehicle type, each with its own [[Price Rule]] and [[Allocation Rule]].

### Schema

| Field | Notes |
|---|---|
| `quotation_id` | FK |
| `service_id` | FK |
| `vehicle_type_id` | **NEW.** FK to vehicle type catalog. Part of unique key. |
| `price_rule_id`, `allocation_rule_id` | Existing FKs kept. |

Unique constraint: `UNIQUE (quotation_id, service_id, vehicle_type_id)`.

### Order-time selection

When the operator picks a Quotation Service for a new [[Order Line]]:

1. Operator picks Service.
2. Operator picks vehicle type from the supported set for that Service in this Quotation.
3. The matching `(quotation, service, vehicle_type)` row resolves to the PriceRule and AllocationRule snapshotted onto the OrderLine.

If only one vehicle type is offered for a Service, step 2 collapses to default selection.

### Cost rates side

Vehicle-type-specific cost rates already live in the Cost Ledger via [[Cost Per Km]] computed per vehicle (and rolled up by vehicle type for cold-start). No change there. The pricing-side vehicle dimension and the cost-side vehicle entity now align.

### What this does *not* change

- One-PriceRule-per-row and one-AllocationRule-per-row.
- Snapshot-on-OrderLine semantics ([[ADR-007 — PriceRule Mirrors CostRule]]).
- The Service catalog itself — Service does **not** carry vehicle type.

## Consequences

- **Good:** Per-vehicle-type pricing is first-class; no Service duplication.
- **Good:** Operator UI is natural — pick Service, then vehicle type.
- **Good:** Aligns with cost-side vehicle modelling.
- **Tradeoff:** Catalog grows by a factor of supported vehicle types per Service. Acceptable — these are real distinct price points.
- **Tradeoff:** Quotation amendment must specify which `(service, vehicle_type)` row is being amended, not just the Service. See [[ADR-024 — Quotation Amendment After In-Progress]].

## Related

- [[Quotation]]
- [[Quotation Service]]
- [[Service]]
- [[Price Rule]]
- [[Allocation Rule]]
- [[Cost Per Km]]
- [[Model — Order to Work Order Chain]]
- [[ADR-007 — PriceRule Mirrors CostRule]]
- [[ADR-016 — Cost Per Km Buckets]]
