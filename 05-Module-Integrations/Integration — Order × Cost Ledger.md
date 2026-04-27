# Integration — Order × Cost Ledger

#integration #module/order-mgmt #module/work-order #module/cost-ledger #status/active

> How Cost Ledger feeds Work Order cost, and how cost + revenue produce route profitability.

## Overview

The [[Cost Ledger]] provides cost rates for vehicles and drivers. [[Work Order]] uses these rates to compute its trip cost. Order revenue (from [[Price Rule]]) and Work Order cost (from Cost Ledger) together give route profitability — in the operations view.

This replaces the old "Delivery Costing" framing. The entity formerly called Delivery is now Work Order.

## Cost Ledger → Work Order

```
work_order.cost =
    fixed_cost_share
  + variable_cost
  + driver_cost
  + events_cost
```

### Fixed Cost Share

```
vehicle_daily_fixed = Σ active Cost Rules for vehicle on that date
                      where classification = fixed_time
                      (depreciation, insurance, registration, etc.)

If vehicle runs multiple Work Orders that day:
  fixed_cost_share = vehicle_daily_fixed
                   × (wo.actual_distance_km / total_vehicle_km_that_day)

If vehicle runs one Work Order that day:
  fixed_cost_share = vehicle_daily_fixed
```

### Variable Cost

Per [[ADR-016 — Cost Per Km Buckets]], variable cost is computed across multiple buckets:

```
variable_cost =
    Σ over buckets (fuel, maintenance, tire, ...):
        cost_per_km(vehicle, bucket, as_of=wo.date) × wo.actual_distance_km
```

Each `cost_per_km(vehicle, bucket)` is smoothed independently from `variable_usage` Cost Events. See [[Cost Per Km]] and [[ADR-014 — Cost Classification]].

### Driver Cost

A Work Order may carry one main driver and any number of helpers via [[Work Order Driver]]. Driver cost sums across all rows:

```
driver_cost = Σ over WorkOrderDriver(wo):
    salary_share(driver, wo)
  + per_trip_rate(driver, role, route, customer)
  + per_km_rate (driver, role, route, customer) × wo.actual_distance_km
```

- `salary_share` — duration-based share of monthly salary per [[ADR-019 — Driver Salary Allocation by Duration Share]]:
  ```
  salary_share = (DriverSalary.amount / days_in_month(wo.date))
               × (wo.duration_seconds / standard_working_seconds_per_day)
  ```
  where `wo.duration_seconds = wo.work_ended_at - wo.driving_started_at`.
- `per_trip_rate` and `per_km_rate` — resolved from [[Driver Rate]] by `(role, scope)`, most-specific-wins (customer > route > default).
- `role` is `main` or `helper`, taken from the [[Work Order Driver]] row.

See [[Integration — Driver × Cost Ledger]] for the full per-driver formula and lifecycle handling.

### Events Cost (per-WO incidentals)

Per [[ADR-015 — Per-WO Cost Events]]:

```
events_cost = Σ CostEvent.amount
              WHERE work_order_id = this WO
                AND classification = one_off
```

Captures tolls, parking fees, port pass fees, customer-specific extras (forklift, document fees) tied to a specific trip. Negative amounts (refunds attributed to a WO) reduce events_cost — see [[ADR-018 — Negative Amounts and Disposal]].

## Cost Ledger API

```
get_rules(date, entity_type=vehicle, entity_id=X, classification=fixed_time)
                                                              → active fixed cost rules
get_rules(date, entity_type=driver,  entity_id=Y, cost_type=salary)
                                                              → monthly salary rule for driver
get_driver_rate(driver, role, basis, route, customer, date)   → per-trip / per-km rate
get_cost_per_km(vehicle_id, bucket, as_of_date)               → smoothed variable rate per bucket
get_events(work_order_id=Z)                                   → per-WO incidentals
```

`entity_type=employee` is deprecated — see [[ADR-013 — Driver as Cost Entity, Not Employee]].

## Revenue vs Cost (Ops View)

```
work_order.display_revenue  ← Σ order_location.allocated_share for done drops
work_order.cost             ← computed from Cost Ledger
work_order.display_margin   = display_revenue − cost
```

**This is the operations view.** See [[ADR-005 — Finance vs Operations Cost Views]] and [[ADR-006 — Revenue Lives on Order, Not Work Order]].

Finance view: `Σ order.billable_revenue` (revenue) vs `Σ CostRule + CostEvent` (cost). Work Order margins do not appear in finance reports.

## Profitability Stack

```
Finance view:
  Revenue: Σ order.billable_revenue by period
  Cost:    Σ CostRule daily + CostEvents by period
  Margin:  Finance revenue − Finance cost

Operations view (per route):
  Revenue: work_order.display_revenue
  Cost:    work_order.cost
  Margin:  work_order.display_margin

Operations view (per order):
  Revenue: order.revenue
  Cost:    Σ work_order.cost × (order's allocated_share / work_order.display_revenue)
           [approximate — for analytics only]
```

## Related

- [[Cost Ledger — Overview]]
- [[Cost Per Km]]
- [[Cost Rule]]
- [[Cost Event]]
- [[Work Order]]
- [[Work Order Driver]]
- [[Driver]]
- [[Driver Salary]]
- [[Driver Rate]]
- [[Route]]
- [[Order]]
- [[Integration — Driver × Cost Ledger]]
- [[ADR-005 — Finance vs Operations Cost Views]]
- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-014 — Cost Classification]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
