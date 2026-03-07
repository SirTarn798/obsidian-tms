# Cost Per Km

#concept #module/cost-ledger #module/vehicle-maintenance

> A derived rate that smooths lumpy maintenance costs into a stable per-kilometer figure for delivery cost allocation.

## Definition

Cost Per Km = Total maintenance/variable spend over a lookback window ÷ Total km driven in that window.

This is **not stored as a fact** — it's computed from [[Cost Event]] data and odometer/trip records. It's recalculated periodically (weekly or monthly), not per-delivery.

## Calculation

```
Inputs:
  - vehicle_id
  - lookback_window (e.g., 6 months)
  - cost_types to include (e.g., ["repair", "maintenance", "tire_replacement"])

Steps:
  1. Sum all CostEvents for this vehicle within the lookback window
     where cost_type in the included types
  2. Get total km driven by this vehicle in the same window
     (from odometer readings or sum of delivery distances)
  3. cost_per_km = total_cost / total_km

Example:
  Last 6 months: 180,000 in maintenance, 28,000 km driven
  Cost/km = 180,000 / 28,000 = 6.43/km
```

## Why a Lookback Window

Maintenance is lumpy — a 40,000 repair in one month followed by nothing for 3 months. Using only the current month would give wildly different cost/km rates. A 6-month window absorbs these spikes and produces a stable rate.

The window length is a tradeoff:

- Too short (1 month): rate swings with every repair
- Too long (2 years): rate is slow to reflect real changes (aging vehicle costs more)
- Sweet spot: 3–6 months for most fleets

## What Costs Are Included

**Included in cost/km:** repairs, scheduled maintenance, tire replacements, parts — anything that scales roughly with usage.

**NOT included in cost/km (these are fixed costs):** depreciation (see [[ADR-004 — Vehicle Purchase Depreciation Spread]]), insurance, registration — these exist regardless of km driven.

## How It's Used

In the [[Integration — Cost Ledger × Delivery Costing|delivery cost allocation]]:

```
delivery_variable_cost = cost_per_km × delivery_distance_km
```

This is combined with fixed cost allocation to get total delivery cost.

## Related

- [[Cost Event]] — source data for the numerator
- [[Delivery]] — consumer of the rate
- [[Integration — Cost Ledger × Delivery Costing]]
- [[Vehicle Maintenance — Overview]]