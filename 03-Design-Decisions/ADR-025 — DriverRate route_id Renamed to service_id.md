# ADR-025 — DriverRate route_id Renamed to service_id

#decision #module/driver-mgmt #module/order-mgmt #status/active

## Status

Accepted. Field rename on [[Driver Rate]].

## Context

[[Driver Rate]] supports a `scope=route` row type that lets a driver earn a different rate when running a particular route. The field carrying the FK was named `route_id`.

In the new model `route` is overloaded:

- [[Route]] is per-WorkOrder geometry — polyline, planned distance, computed by VRP. It is volatile; a single trip on the same lane can produce many different Routes depending on traffic and merging.
- [[Service]] is the customer's named lane — the stable concept the driver actually thinks of as "the route to BKK." It is what a "route premium" pay rate is meant to attach to.

Tying `DriverRate.route_id` to the volatile [[Route]] entity means:

- The same Service across two days produces different Routes; the rate would resolve only by accident.
- VRP merges break the linkage entirely — a merged WO has one Route stitched across multiple customer Services.
- "Most-specific-wins" resolution becomes brittle; the field that should mean "this customer-named lane" instead points at a transient routing artefact.

The intent of `scope=route` was always **service-scope** in the current vocabulary.

## Decision

**Rename `DriverRate.route_id` to `DriverRate.service_id`.** It is set when `scope=service` (also renamed from `scope=route`) and references [[Service]].

### Schema

| Field | Status | Notes |
|---|---|---|
| `route_id` | **DROPPED** | Replaced. |
| `service_id` | **NEW** | FK to Service. Set when `scope=service`. |
| `scope` enum value `route` | **RENAMED** | → `service`. |

Resolution order is unchanged in shape — most-specific-wins, override not stacking:

```
1. customer_id matches WO's order's customer  → customer-scope rate
2. else service_id matches the OrderLine's service_id of any stop on this WO
                                              → service-scope rate
3. else                                       → default-scope rate
```

Note step 2 reads the **OrderLine's** service_id, not anything on Route. A WO with stops from multiple OrderLines (and hence multiple Services) is handled by the resolver picking the rate per the OrderLine the stop belongs to. If multiple Services on one WO have different service-scope rates for the same driver, the driver's compensation is computed per stop, not per WO. This aligns with the cost-back-to-OrderLine direction in [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]].

### Old data

Out of scope — old DriverRate rows with `route_id` are not migrated; new design starts fresh per the project's no-migration stance.

## Consequences

- **Good:** Pay scope attaches to a stable customer-facing entity; rate resolution is VRP-stable.
- **Good:** Aligns with `Service` being per-customer ([[ADR-020 — Service Is Per-Customer]]), so a `(driver, customer, service)` rate triple is unambiguous.
- **Good:** Fits the cost-back direction — pay can be allocated per OrderLine via per-stop resolution.
- **Tradeoff:** Per-stop resolution is more work than one-rate-per-WO. Acceptable, and only needed when service-scope rates apply.
- **Tradeoff:** The word "route" disappears from compensation vocabulary. UI must rename.

## Related

- [[Driver Rate]]
- [[Service]]
- [[Route]]
- [[Order Line]]
- [[Work Order]]
- [[Work Order Driver]]
- [[Integration — Driver × Cost Ledger]]
- [[Model — Order to Work Order Chain]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-020 — Service Is Per-Customer]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
