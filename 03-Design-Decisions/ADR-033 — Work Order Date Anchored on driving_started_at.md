# ADR-033 — Work Order Date Anchored on driving_started_at

#decision #module/work-order #module/cost-ledger #module/driver-mgmt #status/active

## Status

Accepted. Locks the cross-midnight rule for `WorkOrder.date`.

## Context

[[Work Order]] has a `date` field. A trip that starts at 22:00 on Monday and ends at 03:00 Tuesday — what is the WO's date? Several downstream computations assume one WO maps to one date:

- Driver salary daily share per [[ADR-019 — Driver Salary Allocation by Duration Share]] — the daily salary divisor is keyed on date.
- [[Monthly Operational Cost]] daily reports — fixed cost prorations per day.
- Fleet utilisation reports — "vehicle V worked on these N dates this month."
- Driver Rate effective date resolution — `as_of = wo.date`.

Three options were considered:

- **`date = driving_started_at::date`** — anchor on when work began. Operationally clear. Matches the pay-day intuition (a driver whose shift starts at 22:00 Monday is being paid for Monday).
- **`date = midpoint(driving_started_at, work_ended_at)::date`** — more accurate for utilisation, surprising for pay.
- **Permit splitting.** One WO straddles two `date` rows for cost-allocation. Heavy — every aggregator that touches WO date has to know about split-WO summation.

## Decision

**`WorkOrder.date = driving_started_at::date`.**

The WO is conceptually "the work the driver started today" even if it spilled past midnight. This matches pay-day intuition, daily salary share computation, and operator dashboards. Trips that genuinely belong to two days (an unusually long haul) are still single-WO accounted to the day work began.

### Behaviour by Status

| WO status | `date` semantic |
|---|---|
| `planned` | `date` is operator-set or VRP-derived from `planned_start::date`. May be revised before driving begins. |
| `in_progress` | `date` is locked to `driving_started_at::date` at the moment driving begins. |
| `completed` / `closed_partial` | `date` already locked at start; not recomputed at close. |
| `cancelled` | `date` retains its planned value (no driving started). |

A WO that crosses planned-day boundary before driving starts (operator created it for Monday but the driver only begins at 00:30 Tuesday) realigns its `date` to Tuesday at start. This is a one-shot adjustment at the `planned → in_progress` transition.

### Downstream Impacts

| System | Reads `wo.date` for | Effect of anchor rule |
|---|---|---|
| Driver salary daily share ([[ADR-019]]) | `daily_share(wo.date)` | Driver paid for their start-day; matches roster expectations. |
| [[Driver Rate]] resolution | `as_of = wo.date` | Late-night work uses the rate active on the start-day. |
| [[Monthly Operational Cost]] daily reports | Fixed cost share by date | Trips count toward their start-day. |
| Fleet utilisation by date | Vehicle activity by date | A 23:30–02:00 trip counts as one day's activity, on the start-day. |
| Cost Per Km lookback | km driven by date window | Same start-day attribution. |

### Why Not Midpoint or Splitting

Midpoint is more "fair" for utilisation but produces the un-fixable pay-day surprise — the same driver working the same shift two weeks in a row gets a different daily attribution depending on when they happened to finish.

Splitting (WO straddles two dates) is technically correct for cost allocation but every aggregator becomes more complex. The fidelity gain doesn't justify the complexity for the small fraction of trips that genuinely cross midnight.

## Consequences

- **Good:** One WO → one date. Every downstream computation reads the same anchor.
- **Good:** Pay-day intuition holds. Driver and operator are aligned on "what day was this trip."
- **Good:** Late-running trips don't cause same-day double-attribution if a driver does two trips that night.
- **Tradeoff:** A trip that begins at 23:55 and runs eight hours into the next day is mostly attributed to the start-day, slightly distorting daily utilisation. Acceptable; the distortion averages out across the fleet.
- **Tradeoff:** Driver Rate cutovers that fall on a midnight boundary follow the start-day rate. If a rate change is intended to apply mid-trip, the operator must close the WO at midnight and start a new one — same behaviour as today, made explicit.

## Related

- [[Work Order]]
- [[Driver Salary]] · [[Driver Rate]]
- [[Monthly Operational Cost]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-016 — Cost Per Km Buckets]]
