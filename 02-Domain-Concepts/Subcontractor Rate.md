# Subcontractor Rate

#concept #module/cost-ledger #module/work-order #status/active

> Per-trip cost rate that a [[Subcontractor]] charges the operating tenant, parameterized by vehicle type. Used by VRP at planning time to estimate sub-contract cost; replaced at WO close by the actual `subcontract_payout` [[Cost Event]].

## Definition

A Subcontractor Rate defines what a partner charges per trip for a given vehicle type. Mirrors [[Price Rule]] / [[Cost Rule]] structure — methods (`flat`, `per_km`, `per_stop`, `tiered`), time-versioned via close-and-replace per [[ADR-001 — Close and Replace, Never Mutate]], never mutated.

Used by VRP and quotation costing to evaluate sub-contract trips on the same footing as in-house trips. Per [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]] this is the planning-time analog of the in-house per-WO cost formula.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `subcontractor_rate_id` | string | PK |
| `subcontractor_id` | string | FK to [[Subcontractor]] |
| `vehicle_type_id` | string | FK to [[Vehicle Type]] |
| `method` | enum | See methods below |
| `params` | JSON | Method-specific parameters |
| `effective_from` | date | Inclusive |
| `effective_until` | date? | NULL = open |

**Uniqueness:** at any given date, at most one open rate per `(subcontractor_id, vehicle_type_id, method)` — close the old before opening the new.

## Methods

| Method | Params | Cost formula |
|---|---|---|
| `flat` | `{ amount }` | `amount` |
| `per_km` | `{ rate_per_km, min?, max? }` | `rate_per_km × planned_distance_km` (clamped to `[min, max]` if set) |
| `per_stop` | `{ rate_per_stop, base? }` | `(base ?? 0) + rate_per_stop × count(stops)` |
| `tiered` | `{ tiers: [{up_to_km, rate}, ...] }` | Lookup by `planned_distance_km` band |

`planned_distance_km` is the VRP / [[Route]] estimate for the trip. The tenant pays the contractor a single per-trip amount; this is its computation.

## Resolution at Planning

For a Work Order that VRP would assign to sub-contract slot V (where `V.ownership = sub_contracted`, `V.subcontractor_id = S`, `V.vehicle_type_id = T`):

```
rate     = active SubcontractorRate WHERE subcontractor_id = S
                                      AND vehicle_type_id = T
                                      AS OF wo.date
estimate = rate.evaluate(method, params, planned_distance_km, stop_count)
WO.estimated_subcontract_cost = estimate
```

If no rate exists for `(S, T)`, the slot is excluded from VRP selection until a rate is added (or VRP surfaces a warning, depending on UX preference).

## Replacement at WO Close

The `subcontract_payout` is paid to the contractor as a single negotiated amount, often matching the rate but occasionally differing (volume discount, exception fee). The actual amount is recorded as a [[Cost Event]] with `cost_type = subcontract_payout`, `cost_classification = one_off`, `work_order_id = wo.id`, and lands in `WO.events_cost`.

Per [[ADR-029]], the WO cost on a sub-contract trip at close time is:

```
WO.cost = subcontract_payout CostEvent + actual events_cost (any direct payments)
```

The estimated rate is a planning input. The CostEvent is the truth.

## Close and Replace

Rate changes follow [[ADR-001]]:

```
Existing: per_km / Partner X / 22-wheel / 30 baht/km, effective_from=2026-01-01, effective_until=NULL

Raise to 35 effective 2026-06-01:
  1. UPDATE existing SET effective_until = 2026-05-31
  2. INSERT new SET rate_per_km=35, effective_from=2026-06-01, effective_until=NULL
```

Past WO planning estimates resolve to the historical rate by date. Past actuals are unaffected — they live as CostEvents.

## Held for v2

- **Tiered by stop count or weight in addition to distance.** When partner contracts grow more granular.
- **Customer-scoped overrides.** A partner may charge less for a regular customer's lane. Mirrors the customer-scope path in [[Price Rule]] / [[Driver Rate]].

## Related

- [[Subcontractor]]
- [[Vehicle]] — sub-contract slot the rate applies to
- [[Vehicle Type]] — rate dimension
- [[Cost Event]] — actual per-WO cost on close
- [[Price Rule]] · [[Cost Rule]] — structural analogs
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-007 — PriceRule Mirrors CostRule]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
