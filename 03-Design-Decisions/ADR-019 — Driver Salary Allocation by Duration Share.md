# ADR-019 — Driver Salary Allocation by Duration Share

#decision #module/driver-mgmt #module/cost-ledger #status/active

## Status

Accepted. Replaces the same-day allocation_fraction described in earlier drafts of [[Integration — Driver × Cost Ledger]].

## Context

A driver's monthly base salary contributes to per-Work-Order cost. The original distribution rule was same-day allocation_fraction: if a driver runs N WOs that day, each WO bears 1/N of the daily salary share. Days with zero WOs left the daily share unallocated as "overhead for the day."

This produces noisy per-WO cost:

- A driver who runs 1 WO Monday and 5 WOs Tuesday gets a 5× swing in salary contribution per WO, despite the trips being similar in shape.
- A short 2-hour WO bears the same salary share as a 12-hour overnight WO if they fall on the same day.
- Operators reading per-route P&L cannot tell whether margin pressure comes from rates or from accounting noise.

## Decision

**Driver salary plugs into per-WO cost based on the duration of work — WO duration as a fraction of a standard working day.**

```
DriverSalary.daily_share(date) =
    DriverSalary.amount / days_in_month(date)

WorkOrder.driver_salary_share(driver) =
    DriverSalary.daily_share(wo.date)
    × (wo.duration_seconds / standard_working_seconds_per_day)
```

| Symbol | Meaning |
|---|---|
| `DriverSalary.amount` | Monthly base salary, role-agnostic |
| `wo.duration_seconds` | `(wo.work_ended_at - wo.driving_started_at)` expressed in whole seconds. See [[Work Order]] for the timestamp lifecycle. |
| `standard_working_seconds_per_day` | Configured per tenant; default 28,800 (= 8 × 3,600) |

Duration is stored and compared as integer seconds throughout. Display layers convert to hours / minutes for human-readable views; arithmetic stays in seconds to avoid floating-point drift.

### Examples

| WO duration (sec) | Equivalent | Share of daily salary |
|---|---|---|
| 14,400 | 4 h | 0.5 day |
| 28,800 | 8 h | 1.0 day |
| 43,200 | 12 h | 1.5 day |
| 86,400 | 24 h (overnight) | 3.0 day |

### Multiple WOs same day

Each WO bears its own duration share. A driver who runs a 4h WO and another 4h WO on the same day contributes 0.5 + 0.5 = 1.0 daily share total. A driver who runs only one 4h WO contributes 0.5 daily share to per-WO cost; the remaining 0.5 daily share is **idle**, surfaced separately.

### Idle reporting

```
daily_idle_share(driver, date) =
    DriverSalary.daily_share(date)
    - Σ WorkOrder.driver_salary_share(driver) for that date
```

Aggregated across drivers, this is the fleet-level driver utilization metric. It does **not** flow to per-WO cost or to [[Monthly Operational Cost]] (which is non-transport overhead). It is its own line in fleet utilization dashboards: "driver idle hours" with cost-equivalent value.

If a driver works zero WOs in a day, the entire daily share is idle. Over a full month with no WOs, the full salary is idle.

### Live (in-flight) WOs

For WOs in `in_progress` status, use estimated duration from [[Route]] (`estimated_duration_min × 60` to convert to seconds) for live dashboards. Finalize `wo.duration_seconds` from actual timestamps at WO close. Per-WO cost recomputes once on close — consistent with [[ADR-002 — Compute on Read, Cache When Needed]].

### Precision

Duration is integer seconds. No rounding heuristic needed — `duration_seconds / standard_working_seconds_per_day` produces a stable decimal share. Display layers may format as `Hh Mm` for humans; arithmetic stays in seconds.

## DriverSalary as a Concept

DriverSalary is a domain concept with its own concept doc — [[Driver Salary]]. Storage uses [[Cost Rule]] with locked fields:

```
CostRule(
    entity_type           = driver,
    entity_id             = driver_id,
    cost_type             = salary,
    cost_classification   = fixed_time,
    amount                = (DriverSalary.amount),
    frequency             = monthly,
    proration_strategy    = daily_prorate,
    effective_from        = (DriverSalary.effective_from),
    effective_until       = (DriverSalary.effective_until)
)
```

The concept doc owns the field list, lifecycle, and per-WO allocation rule. CostRule is the unified storage primitive. Frontend APIs operate on DriverSalary; ledger queries see the underlying CostRule rows. This preserves the centralized ledger goal while giving DriverSalary a clean domain home.

## Consequences

- **Good:** Per-WO salary contribution scales with actual work duration
- **Good:** Overtime falls out naturally (>1.0 daily share on long WOs)
- **Good:** Overnight / multi-day WOs bear proportional salary load
- **Good:** Idle time is a first-class operational metric, not a hidden bucket
- **Good:** Same-day allocation_fraction (and its noise) is removed
- **Tradeoff:** Requires reliable WO start/end timestamps. Drivers must record both.
- **Tradeoff:** `standard_working_hours_per_day` is a config knob — settable per tenant.
- **Tradeoff:** Live dashboards use estimated duration; finalize at WO close.

## Related

- [[Driver]]
- [[Driver Salary]]
- [[Driver Rate]]
- [[Work Order]]
- [[Work Order Driver]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — Order × Cost Ledger]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-002 — Compute on Read, Cache When Needed]]
- [[ADR-003 — Month-by-Month Proration]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
