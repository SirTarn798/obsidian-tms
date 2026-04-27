# Driver Salary

#concept #module/driver-mgmt #module/cost-ledger #status/active

> Monthly base retainer paid to a [[Driver]]. Allocated to [[Work Order]] cost by duration share. Storage backed by [[Cost Rule]].

## Definition

DriverSalary is the driver's monthly base pay — the retainer that exists regardless of trip activity. It is role-agnostic: a driver who works as `helper` on some WOs and `main` on others draws the same salary. Role-dependent compensation lives on [[Driver Rate]] (per-trip and per-km).

DriverSalary is the domain concept; storage is a [[Cost Rule]] with locked fields. Operators interact with DriverSalary as a domain entity through frontend APIs; ledger queries see the underlying CostRule rows.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `driver_salary_id` | string | Unique identifier |
| `driver_id` | string | Linked [[Driver]] |
| `amount` | decimal | Monthly amount in baht |
| `effective_from` | date | Inclusive start |
| `effective_until` | date? | Inclusive end; NULL = open |
| `created_at` | datetime | Audit |
| `created_by` | string | Audit |

## Storage Mapping

The corresponding [[Cost Rule]] is stored as:

```
CostRule(
    entity_type         = driver,
    entity_id           = driver_id,
    cost_type           = salary,
    cost_classification = fixed_time,
    amount              = DriverSalary.amount,
    frequency           = monthly,
    proration_strategy  = daily_prorate,
    effective_from      = DriverSalary.effective_from,
    effective_until     = DriverSalary.effective_until
)
```

These fields are locked — DriverSalary creation enforces them. An operator cannot create a DriverSalary with `frequency=daily` or `entity_type=user`.

## Lifecycle

Driver salaries follow [[ADR-001 — Close and Replace, Never Mutate]].

### Driver hired with salary

```
INSERT DriverSalary(
    driver_id      = DRV-001,
    amount         = 30000,
    effective_from = 2026-04-15
)
→ INSERT CostRule with locked fields per the storage map above
```

### Salary change (raise or adjustment)

```
1. UPDATE DriverSalary SET effective_until = 2026-05-31
2. INSERT new DriverSalary(amount = 35000, effective_from = 2026-06-01)
   Both operations apply to the underlying CostRule:
3. UPDATE CostRule SET effective_until = 2026-05-31
4. INSERT new CostRule with the new amount
```

### Driver leaves

```
UPDATE DriverSalary SET effective_until = leave_date
→ Underlying CostRule effective_until updated to match
```

### Driver rehired

```
INSERT new DriverSalary (do not reopen old one)
→ INSERT new CostRule
```

The gap between the old `effective_until` and the new `effective_from` correctly produces zero salary cost — no proration, no re-opening.

## At Most One Open Salary

A driver has at most one open DriverSalary at any time. Enforced by the at-most-one-open-rule constraint on the underlying [[Cost Rule]] for `(entity_type=driver, entity_id=X, cost_type=salary)`. Attempting to create a second open DriverSalary fails with a clear error directing the operator to close the existing one first.

## Allocation to Work Orders

Per [[ADR-019 — Driver Salary Allocation by Duration Share]]:

```
WO_salary_share(driver, wo) =
    (DriverSalary.amount / days_in_month(wo.date))
    × (wo.duration_seconds / standard_working_seconds_per_day)
```

Duration is stored and computed as integer seconds: `wo.work_ended_at − wo.driving_started_at`. `standard_working_seconds_per_day` defaults to 28,800 (8 hours), configurable per tenant. See [[Work Order]] for the timestamp lifecycle.

This contributes to `WorkOrder.driver_cost`:

```
WorkOrder.driver_cost = Σ over WorkOrderDriver(wo):
    WO_salary_share(driver, wo)
  + per_trip_rate(driver, role, route, customer)
  + per_km_rate (driver, role, route, customer) × wo.actual_distance_km
```

`per_trip_rate` and `per_km_rate` are resolved from [[Driver Rate]] by role and scope.

## Idle Time

Daily salary share that is not allocated to any WO is **idle**. Idle is reported as a fleet utilization metric, separately from per-WO cost and from [[Monthly Operational Cost]] (which is non-transport overhead). See [[ADR-019 — Driver Salary Allocation by Duration Share]].

```
idle_share(driver, date) =
    daily_share(driver, date)
    - Σ WO_salary_share(driver, wo) for wo on that date
```

Idle is informational. It is not a billable or allocable cost — just visibility into how much of the salary the fleet is not yet capturing in trip activity.

## Why This Lives in the Cost Ledger

Storing DriverSalary as a [[Cost Rule]] (rather than in a separate `driver_salary` table) preserves the centralized ledger goal. All cost-related queries — monthly totals, per-driver history, finance vs operations views — see DriverSalary alongside vehicle depreciation, fuel events, and office salary, with a single query interface.

The DriverSalary concept exists at the domain layer for clean operator workflows; the CostRule storage exists for centralized querying.

## Related

- [[Driver]]
- [[Driver Rate]]
- [[Cost Rule]]
- [[Work Order]]
- [[Work Order Driver]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-003 — Month-by-Month Proration]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-014 — Cost Classification]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[Integration — Driver × Cost Ledger]]
