# Integration — Driver × Cost Ledger

#integration #module/driver-mgmt #module/cost-ledger #status/active

> How [[Driver]] lifecycle and compensation rates feed the [[Cost Ledger]] and produce per-Work-Order driver cost.

## Overview

Drivers are the only humans tracked as transport-direct cost entities — see [[ADR-013 — Driver as Cost Entity, Not Employee]]. Office staff (controllers, planners, accountants) are tracked separately via [[Integration — User × Cost Ledger]] as overhead.

A driver's cost on a Work Order is the sum of three components:

```
driver_cost(wo, driver, role) =
    salary_share(driver, wo)                                  -- duration-based share of monthly salary
  + per_trip_amount(driver, role, wo)                         -- flat per-trip rate
  + per_km_amount(driver, role, wo)                           -- per-km rate × actual distance
```

`work_order.driver_cost` is the sum across all rows of [[Work Order Driver]] for that WO (main + helpers).

## Cost Sources

| Component | Stored as | Concept | Role-aware |
|---|---|---|---|
| Monthly base salary | [[Cost Rule]] with locked fields (`entity_type=driver, cost_type=salary, classification=fixed_time, frequency=monthly, proration=daily_prorate`) | [[Driver Salary]] | No |
| Per-trip pay | [[Driver Rate]] `(role, basis=per_trip, scope)` | [[Driver Rate]] | Yes |
| Per-km pay | [[Driver Rate]] `(role, basis=per_km, scope)` | [[Driver Rate]] | Yes |

### Why monthly is role-agnostic

Monthly salary represents the employment relationship, not the per-WO duty. A driver who works as helper on some WOs and main on others draws the same monthly paycheck. Role-dependent variation lives in per-trip and per-km rates only.

## Salary Share by Duration

Per [[ADR-019 — Driver Salary Allocation by Duration Share]]:

```
daily_share(driver, date) =
    DriverSalary.amount / days_in_month(date)

salary_share(driver, wo) =
    daily_share(driver, wo.date)
    × (wo.duration_seconds / standard_working_seconds_per_day)

where:
    wo.duration_seconds = wo.work_ended_at - wo.driving_started_at
    standard_working_seconds_per_day defaults to 28,800 (= 8 × 3,600)
```

Examples (driver salary 30,000/month, May = 31 days, daily share = 967.74):

| WO duration | Salary share |
|---|---|
| 4 hours (14,400 s) | 483.87 |
| 8 hours (28,800 s) | 967.74 |
| 12 hours (43,200 s) | 1,451.61 |

### Idle time

Daily share that is not allocated to any WO is **idle**:

```
daily_idle(driver, date) =
    daily_share(driver, date)
  - Σ salary_share(driver, wo) for wo with wo.date = date
```

Idle is reported as a fleet utilization metric, separately from per-WO cost and from [[Monthly Operational Cost]] (which is non-transport overhead). It is its own line in fleet utilization dashboards.

## Lifecycle Events → Cost Rules

### Driver hired with monthly salary

```
Event: DRV-002 hired on 2026-04-15, salary 30,000/month, per-trip main 500, per-trip helper 200

Actions:
  1. INSERT DriverSalary(driver_id=DRV-002, amount=30000, effective_from=2026-04-15)
     → INSERT CostRule(entity_type=driver, entity_id=DRV-002,
                       cost_type=salary, classification=fixed_time,
                       amount=30000, frequency=monthly,
                       proration_strategy=daily_prorate,
                       effective_from=2026-04-15)
  2. INSERT DriverRate(driver=DRV-002, role=main,   basis=per_trip, scope=default, rate=500, effective_from=2026-04-15)
  3. INSERT DriverRate(driver=DRV-002, role=helper, basis=per_trip, scope=default, rate=200, effective_from=2026-04-15)

Result: April salary auto-prorated to 16/30 days = 16,000 baht (monthly only,
        before any per-WO activity)
```

### Salary change — close and replace

Per [[ADR-001 — Close and Replace, Never Mutate]] and the at-most-one-open-rule constraint:

```
Event: DRV-002 raise to 35,000 effective 2027-04-01

Actions:
  1. UPDATE DriverSalary SET effective_until=2027-03-31  (closes underlying CostRule)
  2. INSERT new DriverSalary(amount=35000, effective_from=2027-04-01)
     → INSERT new CostRule with effective_from=2027-04-01

Past Work Orders resolve to the historical salary by date.
```

### Per-km / per-trip rate change

Same pattern, on [[Driver Rate]]:

```
Event: DRV-002 main per-km default rate raised from 2.5 → 3.0 effective 2027-05-01

Actions:
  1. UPDATE DriverRate SET effective_until=2027-04-30 WHERE rate_id=...
  2. INSERT DriverRate(... rate=3.0, effective_from=2027-05-01)
```

### Driver leaves

```
Event: DRV-002 leaves on 2027-06-20

Actions:
  1. UPDATE Driver SET status=inactive
  2. UPDATE DriverSalary SET effective_until=2027-06-20
  3. UPDATE all open DriverRate rows SET effective_until=2027-06-20
  4. (optional) Soft-disable User if mobile login was enabled

Result: June salary auto-prorated to 20/30 days
```

### Driver rehired — new rule, do not reopen

```
Event: DRV-002 rehired on 2027-10-01 at 38,000

Actions:
  1. UPDATE Driver SET status=active
  2. INSERT new DriverSalary(amount=38000, effective_from=2027-10-01)
  3. INSERT new DriverRate rows for active rates

Result: Gap from June 20 to Oct 1 correctly shows zero cost
```

## Per-WO Calculation

For each Work Order at close, iterate [[Work Order Driver]] rows:

```
for each (wo, driver, role) in WorkOrderDriver:
    salary_rule  = active CostRule for driver on wo.date with cost_type=salary
    salary_share = (salary_rule.amount / days_in_month(wo.date))
                 × (wo.duration_seconds / standard_working_seconds_per_day)

    per_trip = resolve_rate(driver, role, basis=per_trip, route=wo.route, customer=wo.customer)
    per_km   = resolve_rate(driver, role, basis=per_km,   route=wo.route, customer=wo.customer)

    contribution = salary_share
                 + per_trip
                 + per_km × wo.total_distance_actual_km

work_order.driver_cost = Σ contribution
```

`resolve_rate` follows most-specific-wins precedence in [[Driver Rate]] (customer > route > default).

For WOs `in_progress`, live dashboards may estimate `wo.duration_seconds` from [[Route]]; canonical value is set at WO close per [[ADR-002 — Compute on Read, Cache When Needed]].

## Realtime Compensation Endpoint (Mobile)

The driver mobile app shows live accrued earnings:

```
GET /driver/me/compensation?from=YYYY-MM-DD&to=YYYY-MM-DD
→ {
    salary_share_total:  <Σ salary_share across completed WOs in window>,
    per_trip_total:      <Σ per-trip across completed WOs>,
    per_km_total:        <Σ per-km × actual_km across completed WOs>,
    breakdown: [
       { wo_id, role, basis, rate, units, amount, accrued_at }, ...
    ]
  }
```

**Accrual triggers:**

- Per-trip and per-km accrue when `WorkOrder.status` advances to `completed` or `closed_partial`.
- Salary share accrues at the same trigger — once `wo.duration_seconds` is final.
- Idle hours appear in operator dashboards but not in driver compensation (the driver receives the full monthly salary regardless of idle).

## Cost Ledger API (Driver-side)

```
get_rules(date, entity_type=driver, entity_id=X)              → active monthly salary rule
get_driver_rate(driver, role, basis, route, customer, date)   → resolved DriverRate
list_active_drivers(date)                                     → for daily-share allocation
```

## Controller / Office Staff

Controller, planner, accountant, CEO, and other office Users are **not** in this integration. Their salaries are tracked via [[Integration — User × Cost Ledger]] with `entity_type=user, classification=fixed_time`, flowing to [[Monthly Operational Cost]] as overhead. They are never allocated to individual Work Orders.

## Related

- [[Driver]]
- [[Driver Salary]]
- [[Driver Rate]]
- [[Work Order]]
- [[Work Order Driver]]
- [[Cost Ledger — Overview]]
- [[Cost Rule]]
- [[Integration — Driver × Work Order]]
- [[Integration — Order × Cost Ledger]]
- [[Integration — User × Cost Ledger]]
- [[Monthly Operational Cost]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-003 — Month-by-Month Proration]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-014 — Cost Classification]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
