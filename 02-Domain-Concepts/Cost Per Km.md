# Cost Per Km

#concept #module/cost-ledger #module/vehicle-maintenance #status/active

> A derived per-kilometer rate per vehicle, split into buckets (`fuel`, `maintenance`, `tire`) so each cost component is independently visible. Smooths lumpy events into stable rates for per-WO variable cost.

## Definition

Cost Per Km is a derived rate, computed per `(vehicle, bucket)`. It is **not stored as a fact** — computed from [[Cost Event]] data and odometer/trip records. Recalculated weekly or monthly per [[ADR-002 — Compute on Read, Cache When Needed]] and [[ADR-016 — Cost Per Km Buckets]].

## Buckets

| Bucket | Sources | Default lookback |
|---|---|---|
| `fuel` | [[Fuel Log]] events, manual fuel entries | 3 months |
| `maintenance` | Repair events, scheduled service events, parts | 6 months |
| `tire` | Tire purchase events, tire-specific work | 12 months |

Buckets are extensible — new buckets (`lubricant`, `wash`) can be added without schema change. Each bucket has its own lookback window per its volatility profile.

Only events with `cost_classification = variable_usage` feed the buckets. `one_off` events (per-WO incidentals, vehicle purchase, disposal) are excluded.

## Calculation

```
Inputs:
  - vehicle_id
  - bucket (fuel | maintenance | tire | ...)
  - lookback_window (per bucket default, configurable per tenant)

Steps:
  1. Sum CostEvent.amount for this vehicle within the lookback window
     where cost_classification = variable_usage AND cost_type in bucket's types
  2. Sum km driven by this vehicle in the same window
     (from odometer readings on completed Work Orders)
  3. cost_per_km(vehicle, bucket) = total_cost / total_km

Example (vehicle V1, last 6 months):
  fuel events:        420,000 baht, 28,000 km → 15.00/km
  maintenance events: 180,000 baht, 28,000 km →  6.43/km
  tire events:         84,000 baht, 28,000 km →  3.00/km

Total variable cost rate: 24.43/km
```

## Total Variable Cost on a Work Order

```
WO.variable_cost =
    Σ over buckets:
        cost_per_km(vehicle, bucket, as_of=wo.date) × wo.actual_distance_km
```

Equivalent: `(fuel_cpk + maintenance_cpk + tire_cpk) × wo.actual_distance_km`.

## Cold Start

For a brand-new vehicle with no history in the lookback window, cost_per_km falls back to the **per-bucket default on [[Vehicle Type]]** — `default_fuel_per_km`, `default_maintenance_per_km`, `default_tire_per_km`. Locked by [[ADR-032 — Cost Per Km Cold-Start Default from Vehicle Type]].

Resolution per `(vehicle, bucket)`:

```
events_in_window = CostEvents matching (vehicle, bucket) in lookback window
km_in_window     = km driven by vehicle in lookback window

IF events_in_window non-empty AND km_in_window > 0:
    cpk = Σ events.amount / km_in_window      (computed)
ELSE:
    cpk = vehicle.vehicle_type.default_<bucket>_per_km   (seed)
```

Cold-start mode is per-bucket — a new vehicle may have stable fuel data after one month (high event volume) but no maintenance signal yet, in which case `fuel` reads computed and `maintenance` still reads default.

Sub-contracted [[Vehicle]] rows do not consult CPK at all — sub-contract WO cost is shaped by [[Subcontractor Rate]] and the `subcontract_payout` event per [[ADR-029]].

## What Is NOT in Cost Per Km

| Cost | Where it lives instead |
|---|---|
| Depreciation, insurance, registration | `fixed_time` Cost Rules → per-WO `fixed_cost_share` |
| Vehicle purchase | One-off event, cash flow only |
| Tolls, parking, port fees | `one_off` events with `work_order_id` → per-WO `events_cost` ([[ADR-015 — Per-WO Cost Events]]) |
| Driver salary, per-trip, per-km | [[Driver Salary]] + [[Driver Rate]] |
| Office overhead | `fixed_time` rules with `entity_type=user` or `facility` → [[Monthly Operational Cost]] |

## Related

- [[Cost Event]] — source data
- [[Work Order]] — consumer of the rate via `variable_cost`
- [[Fuel Log]] — feeds the `fuel` bucket
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-014 — Cost Classification]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
- [[ADR-032 — Cost Per Km Cold-Start Default from Vehicle Type]]
- [[Integration — Order × Cost Ledger]]
- [[Integration — Vehicle Maintenance × Cost Ledger]]
- [[Integration — Fuel Log × Cost Ledger]]