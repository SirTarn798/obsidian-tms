# ADR-016 — Cost Per Km Buckets

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

[[Cost Per Km]] was originally a single derived rate per vehicle, lumping fuel, maintenance, tire, and repair into one number. This obscures real economics:

- **Fuel** is highly volatile (price swings, route terrain, load weight). Reacts to global oil prices and route mix.
- **Maintenance** is lumpy but periodic (scheduled service intervals, occasional repair). Smooths over months.
- **Tire** is consumable with predictable wear pattern (km-based, not time-based). Replacement events are spiky but per-tire economics is stable.

A single 6-month average hides which component is moving. Pricing decisions ("fuel doubled this quarter, raise rates") need per-component visibility.

## Decision

**Cost Per Km is computed per cost-type bucket. Total variable cost is the sum across buckets.**

```
cost_per_km(vehicle, fuel,        as_of) = Σ fuel events / Σ km in window
cost_per_km(vehicle, maintenance, as_of) = Σ maintenance events / Σ km in window
cost_per_km(vehicle, tire,        as_of) = Σ tire events / Σ km in window

variable_cost(WO) = (fuel_cpk + maintenance_cpk + tire_cpk) × WO.actual_distance_km
```

### Initial buckets

| Bucket | Sources | Default lookback |
|---|---|---|
| `fuel` | [[Fuel Log]] events, manual fuel entries | 3 months |
| `maintenance` | Repair events, scheduled service events, parts | 6 months |
| `tire` | Tire purchase events, tire-specific work | 12 months |

Different windows per bucket reflect different volatility profiles. Defaults are configurable per tenant.

### Bucket selection by classification

Only events with `cost_classification = variable_usage` feed the buckets ([[ADR-014 — Cost Classification]]). `one_off` events (e.g. accident repair attributed to a WO, vehicle purchase) are excluded — those are per-WO incidentals via [[ADR-015 — Per-WO Cost Events]] or vehicle disposal/write-off via [[ADR-018 — Negative Amounts and Disposal]].

### Extensibility

Adding a new bucket (`lubricant`, `wash`) is a config change, not a schema change. Each cost_type maps to one bucket; the mapping is stored alongside `cost_classification`. Unbucketed `variable_usage` cost types can be assigned to a default bucket or surfaced as a config error.

### Cold start

For a brand-new vehicle with insufficient km in the lookback window, `cost_per_km` falls back to:

1. Fleet-type average for the same vehicle type (if available)
2. A manually-entered baseline (operator override)
3. Zero (last resort, with UI warning)

The cold-start mode is per-bucket. A new vehicle may have stable fuel data after 1 month (high fuel event volume) but no maintenance signal yet.

### Reporting

Per-bucket rates are surfaced in the vehicle dashboard. A spike in `fuel_cpk` is immediately visible without confounding with maintenance trends. The total `cost_per_km` is also shown for backwards compatibility with single-rate consumers.

## Consequences

- **Good:** Each cost component is independently visible and trackable
- **Good:** Operators can react to price/wear signals per component
- **Good:** Smoothing window can be tuned per bucket (fuel volatility ≠ tire volatility)
- **Good:** Cold-start fallbacks are explicit, not silent
- **Tradeoff:** More compute per dashboard query (acceptable at fleet scale, with caching per [[ADR-002 — Compute on Read, Cache When Needed]])
- **Tradeoff:** New cost types must be assigned to a bucket on creation

## Related

- [[Cost Per Km]]
- [[Cost Event]]
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-014 — Cost Classification]]
- [[Integration — Fuel Log × Cost Ledger]]
- [[Integration — Vehicle Maintenance × Cost Ledger]]
