# ADR-003 — Month-by-Month Proration

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

When an employee is paid 60,000/month and we query "cost for Feb 1–2" vs "cost for Dec 1–2", should the result be the same?

**No.** February has 28 days, December has 31. The daily rate differs:

- Feb: 60,000 ÷ 28 = 2,142.86/day → 2 days = **4,285.71**
- Dec: 60,000 ÷ 31 = 1,935.48/day → 2 days = **3,870.97**

This is not a rounding issue — it's a 10.7% difference. Getting this wrong means every partial-month calculation is inaccurate.

## Decision

**All date range queries are split into month chunks before computation.** The algorithm:

1. Break `[start_date, end_date]` into: `[partial_first_month, full_months..., partial_last_month]`
2. For each chunk, use that month's actual `days_in_month` for proration
3. For each rule overlapping a chunk: `daily_rate = monthly_amount / days_in_month`, then `cost = daily_rate × active_days_in_chunk`
4. Sum across all chunks

This handles all edge cases naturally:

- Employee joins mid-month → first chunk is partial
- Employee leaves mid-month → last chunk is partial
- Query spans Feb → daily rate uses 28 (or 29 in leap year)
- Full month → active_days = days_in_month, so cost = exact monthly amount

## Proration Strategies

Each [[Cost Rule]] has a `proration_strategy` field:

- `daily_prorate` — as described above (default for salaries, maintenance)
- `full_period` — charge full monthly amount if any day overlaps (used for insurance, subscriptions)
- `none` — for daily-frequency rules where proration doesn't apply

## Consequences

- **Good:** Mathematically correct for all month lengths, including leap years
- **Good:** Partial month queries (common for new hires, vehicle purchases) are accurate
- **Good:** Full month queries return the exact monthly amount (no floating point drift)
- **Tradeoff:** Slightly more complex computation than naive `amount / 30 * days`, but this is the correct approach

### Code Example
```
"""
Cost computation engine.

The key design decision: we compute costs MONTH BY MONTH.

Why? Because for monthly-paid employees:
  - Daily rate in Feb (28 days) = salary / 28
  - Daily rate in Dec (31 days) = salary / 31
  - So "1-2 Feb" costs MORE per day than "1-2 Dec"

Algorithm for a date range query:
  1. Split the range into month chunks: [partial_first_month, full_months..., partial_last_month]
  2. For each month chunk, find overlapping rules
  3. Compute cost per rule using the correct days_in_month for proration
  4. Sum one-time events in the range
"""

import calendar
from dataclasses import dataclass, field
from datetime import date
from typing import List, Optional

from models import CostRule, CostEvent, Frequency, ProrationStrategy


@dataclass
class MonthlyCostBreakdown:
    year: int
    month: int
    days_in_month: int
    active_days: int        # how many days in this month are within the query range
    rule_costs: float       # total from ongoing rules
    event_costs: float      # total from one-time events
    details: list = field(default_factory=list)  # per-rule/event detail

    @property
    def total(self) -> float:
        return self.rule_costs + self.event_costs


@dataclass
class CostDetail:
    entity_type: str
    entity_id: str
    cost_type: str
    amount: float
    computation_note: str   # human-readable explanation of how we got this number


@dataclass
class CostResult:
    start_date: date
    end_date: date
    total: float
    monthly_breakdown: List[MonthlyCostBreakdown]
    entity_type_filter: Optional[str] = None


def _month_ranges(start: date, end: date):
    """
    Yields (month_start, month_end, year, month) tuples covering the range.
    The first and last months may be partial.
    """
    current = start
    while current <= end:
        year, month = current.year, current.month
        _, days_in_month = calendar.monthrange(year, month)
        month_last_day = date(year, month, days_in_month)

        chunk_end = min(month_last_day, end)
        yield current, chunk_end, year, month, days_in_month
        # move to first day of next month
        if month == 12:
            current = date(year + 1, 1, 1)
        else:
            current = date(year, month + 1, 1)


def compute_rule_cost_for_month(
    rule: CostRule,
    month_start: date,
    month_end: date,
    days_in_month: int,
) -> Optional[CostDetail]:
    """
    Compute the cost contribution of a single rule for a (possibly partial) month.

    For MONTHLY frequency with DAILY_PRORATE:
      daily_rate = amount / days_in_month
      active_days = overlap between [rule.effective_from..rule.effective_until]
                    and [month_start..month_end]
      cost = daily_rate * active_days

    This correctly handles:
      - Employee joins mid-month
      - Employee leaves mid-month
      - Query range is partial month
      - Feb vs Dec daily rate difference
    """
    if not rule.overlaps(month_start, month_end):
        return None

    # Clamp rule effective dates to the month chunk
    overlap_start = max(rule.effective_from, month_start)
    overlap_end = min(
        rule.effective_until if rule.effective_until else month_end,
        month_end,
    )
    active_days = (overlap_end - overlap_start).days + 1  # inclusive

    if active_days <= 0:
        return None

    if rule.frequency == Frequency.MONTHLY:
        if rule.proration_strategy == ProrationStrategy.DAILY_PRORATE:
            daily_rate = rule.amount / days_in_month
            cost = daily_rate * active_days
            note = (
                f"{rule.amount}/mo ÷ {days_in_month} days × {active_days} active days "
                f"({overlap_start} to {overlap_end})"
            )
        elif rule.proration_strategy == ProrationStrategy.FULL_PERIOD:
            cost = rule.amount
            note = f"{rule.amount}/mo full period charge"
        else:
            daily_rate = rule.amount / days_in_month
            cost = daily_rate * active_days
            note = f"{rule.amount}/mo ÷ {days_in_month} days × {active_days} days (default prorate)"

    elif rule.frequency == Frequency.DAILY:
        cost = rule.amount * active_days
        note = f"{rule.amount}/day × {active_days} days"

    elif rule.frequency == Frequency.YEARLY:
        if rule.proration_strategy == ProrationStrategy.DAILY_PRORATE:
            year = month_start.year
            days_in_year = 366 if calendar.isleap(year) else 365
            daily_rate = rule.amount / days_in_year
            cost = daily_rate * active_days
            note = f"{rule.amount}/yr ÷ {days_in_year} days × {active_days} days"
        else:
            # Full period: charge proportional to month
            cost = rule.amount / 12
            note = f"{rule.amount}/yr ÷ 12 months"
    else:
        return None

    return CostDetail(
        entity_type=rule.entity_type.value,
        entity_id=rule.entity_id,
        cost_type=rule.cost_type,
        amount=round(cost, 2),
        computation_note=note,
    )


def compute_costs(
    rules: List[CostRule],
    events: List[CostEvent],
    start: date,
    end: date,
    entity_type: Optional[str] = None,
    entity_id: Optional[str] = None,
) -> CostResult:
    """
    Compute total costs for a date range with monthly breakdown.

    Filters by entity_type and/or entity_id if provided.
    """
    # Filter rules and events
    filtered_rules = rules
    filtered_events = events

    if entity_type:
        filtered_rules = [r for r in filtered_rules if r.entity_type.value == entity_type]
        filtered_events = [e for e in filtered_events if e.entity_type.value == entity_type]
    if entity_id:
        filtered_rules = [r for r in filtered_rules if r.entity_id == entity_id]
        filtered_events = [e for e in filtered_events if e.entity_id == entity_id]

    monthly_breakdowns = []

    for month_start, month_end, year, month, days_in_month in _month_ranges(start, end):
        active_days = (month_end - month_start).days + 1
        details = []
        rule_total = 0.0

        # Compute rule costs for this month chunk
        for rule in filtered_rules:
            detail = compute_rule_cost_for_month(rule, month_start, month_end, days_in_month)
            if detail:
                details.append(detail)
                rule_total += detail.amount

        # Sum one-time events in this month chunk
        event_total = 0.0
        for event in filtered_events:
            if month_start <= event.occurred_on <= month_end:
                details.append(CostDetail(
                    entity_type=event.entity_type.value,
                    entity_id=event.entity_id,
                    cost_type=event.cost_type,
                    amount=event.amount,
                    computation_note=f"One-time event on {event.occurred_on}",
                ))
                event_total += event.amount

        monthly_breakdowns.append(MonthlyCostBreakdown(
            year=year,
            month=month,
            days_in_month=days_in_month,
            active_days=active_days,
            rule_costs=round(rule_total, 2),
            event_costs=round(event_total, 2),
            details=details,
        ))

    total = sum(mb.total for mb in monthly_breakdowns)

    return CostResult(
        start_date=start,
        end_date=end,
        total=round(total, 2),
        monthly_breakdown=monthly_breakdowns,
        entity_type_filter=entity_type,
    )
```