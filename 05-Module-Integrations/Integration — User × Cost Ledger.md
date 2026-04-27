# Integration — User × Cost Ledger

#integration #module/driver-mgmt #module/cost-ledger #status/active

> How office staff salaries (controllers, planners, accountants, owners) are tracked in the [[Cost Ledger]] without being allocated to per-Work-Order cost.

## Overview

Per [[ADR-013 — Driver as Cost Entity, Not Employee]], only [[Driver]]s are charged into per-Work-Order cost. Office staff salaries are still real business costs and **are tracked in the [[Cost Ledger]]** — but they flow into [[Monthly Operational Cost]] (overhead bucket), not into individual Work Orders.

Office staff are represented as User rows. Each salaried User may have a [[Cost Rule]] with `entity_type=user`.

## Lifecycle Events → Cost Rules

Same shape as driver salary, different `entity_type`:

### Office staff hired

```
Event: USR-101 hired as planner on 2026-03-01, salary 35,000/month
Action: INSERT CostRule(
   entity_type=user, entity_id=USR-101,
   cost_type=salary, cost_classification=fixed_time,
   amount=35000, frequency=monthly,
   proration_strategy=daily_prorate,
   effective_from=2026-03-01
)
```

The at-most-one-open-rule constraint applies — a second open salary rule for the same user is rejected. See [[ADR-001 — Close and Replace, Never Mutate]].

### Salary change

Per [[ADR-001 — Close and Replace, Never Mutate]]:

```
Event: USR-101 raise to 40,000 effective 2026-09-01
Actions:
  1. UPDATE existing CostRule SET effective_until=2026-08-31
  2. INSERT new CostRule SET amount=40000, effective_from=2026-09-01
```

### Office staff leaves

```
Event: USR-101 leaves on 2026-12-15
Actions:
  1. UPDATE User SET status=disabled
  2. UPDATE CostRule SET effective_until=2026-12-15
Result: December cost auto-prorated to 15/31 days
```

### One-off bonuses, reimbursements, etc.

Use [[Cost Event]] with `entity_type=user`, dated to the relevant day.

## Allocation Behavior

```
allocate_to_wo(entry):
    entry.entity_type ∈ (vehicle, driver) AND classification ≠ overhead-only
                                            → Yes, charge to Work Order
    entry.entity_type ∈ (user, facility)    → No, stay in overhead pool
```

Office staff cost rules are **excluded** from `work_order.cost` aggregation. They are **included** in [[Monthly Operational Cost]].

This is a reporting distinction enforced by the entity_type filter. The cost rules and events live in the same Cost Ledger as everything else; only the rollup logic differs.

## Reporting

The Monthly Operational Cost report breaks office salary out from transport-direct cost:

```
overhead.office_salaries =
    Σ CostRule (entity_type=user, cost_type=salary) prorated to days in month
  + Σ CostEvent (entity_type=user) in month
```

A drill-down per-User view is available for finance to audit individual obligations.

## Why this is not "Employee Management"

Storing salary against a User row does not reintroduce the deprecated Employee module. The User table is an auth record with optional salary metadata — it does not require HR fields (job title, department, contract type, hire formality). A controller hired today needs only:

- Username + password for login
- Role for permissions
- Optional CostRule for salary (if salaried)

That is a far thinner record than the old Employee. See [[ADR-013 — Driver as Cost Entity, Not Employee]] for the full rationale.

## Cost Ledger API

```
get_rules(date, entity_type=user, entity_id=Z)   → active office salary rule
list_active_users(date)                           → for monthly reporting
```

## Related

- [[Monthly Operational Cost]]
- [[Cost Ledger — Overview]]
- [[Cost Rule]]
- [[Cost Event]]
- [[Integration — Driver × Cost Ledger]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-003 — Month-by-Month Proration]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
