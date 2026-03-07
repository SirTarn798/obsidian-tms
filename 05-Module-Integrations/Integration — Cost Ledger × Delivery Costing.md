# Integration — Cost Ledger × Delivery Costing

#integration #module/cost-ledger #module/order-mgmt #status/active

> How the cost ledger feeds into per-delivery cost calculation.

## Overview

This is the bridge between "how much does our fleet cost?" (finance question) and "how much did this specific delivery cost?" (operations question). See [[ADR-005 — Finance vs Operations Cost Views]] for why these are separate.

## Delivery Cost Formula

```
delivery_cost =
    fixed_cost_share
  + variable_cost
  + driver_cost
```

### Fixed Cost Share

The vehicle has daily fixed costs from [[Cost Rule]]s: depreciation, insurance, registration, etc.

```
vehicle_daily_fixed = sum of all active fixed-cost rules for the vehicle on that day
```

Allocation method depends on how many deliveries the vehicle does that day:

- **Single delivery that day:** vehicle bears 100% of daily fixed cost
- **Multiple deliveries:** proportional by distance or time

```
fixed_cost_share = vehicle_daily_fixed × (delivery_distance / total_vehicle_distance_that_day)
```

### Variable Cost (Maintenance)

Uses the [[Cost Per Km]] rate, which is pre-calculated from maintenance history:

```
variable_cost = cost_per_km × delivery_distance_km
```

The cost/km rate is recalculated weekly/monthly, not per delivery. This smooths lumpy repair costs. See [[Cost Per Km]] for calculation details.

### Driver Cost

Uses the driver's [[Cost Rule]] (salary):

```
If monthly salary with daily prorate:
  driver_daily_rate = monthly_salary / days_in_that_month
  driver_cost = driver_daily_rate × (delivery_hours / working_hours_per_day)

If daily rate:
  driver_cost = daily_rate × (delivery_hours / working_hours_per_day)
```

Note: driver cost per delivery inherits the same Feb-vs-Dec proration from [[ADR-003 — Month-by-Month Proration]].

## Example

```
Delivery A: vehicle-001, emp-001, 30km, 2025-02-15, took 3 hours

Vehicle daily fixed costs:
  depreciation: 547.70/day
  insurance: 3000/mo ÷ 28 days = 107.14/day
  Total: 654.84/day
  Vehicle did 2 deliveries today (80km total)
  Fixed share: 654.84 × (30/80) = 245.57

Variable cost:
  cost_per_km = 6.43 (from last 6 months of maintenance)
  30km × 6.43 = 192.90

Driver cost:
  emp-001 salary: 60,000/mo, Feb has 28 days
  Daily rate: 60,000/28 = 2,142.86
  3 hours of 8-hour day: 2,142.86 × (3/8) = 803.57

Total delivery cost: 245.57 + 192.90 + 803.57 = 1,242.04
```

## Data Flow

```
Vehicle Maintenance Events ──→ Cost Per Km calculation
                                        ↓
Cost Rules (depreciation,    ──→ Delivery Cost ←── Delivery record
  insurance, salary)                Breakdown        (distance, time,
                                        ↓              vehicle, driver)
                               Profitability Analysis
                               (delivery cost vs revenue)
```

## What the Cost Ledger Provides to This Layer

1. `get_rules(date, entity_type=vehicle, entity_id=X)` → active fixed cost rules for a vehicle
2. `get_rules(date, entity_type=employee, entity_id=Y)` → salary rule for the driver
3. `get_events(lookback_window, entity_type=vehicle, entity_id=X, cost_type in [repair, maintenance])` → for cost/km calculation

## Open Questions

- How to handle deliveries that span midnight (two calendar days with different daily rates)?
- Should fuel be a separate variable cost or rolled into cost/km?
- How to allocate fixed costs on days with zero deliveries? (→ idle cost, see [[ADR-005 — Finance vs Operations Cost Views]])

## Related

- [[Cost Ledger — Overview]]
- [[Delivery]]
- [[Cost Per Km]]
- [[Order Management — Overview]]
- [[Work Order Income — Overview]] — revenue side, needed for profitability