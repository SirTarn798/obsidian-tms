# ADR-027 — Helper-Only Mid-Trip Junction Mutation

#decision #module/work-order #module/driver-mgmt #status/active

## Status

Accepted. Refines the freeze rule from [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]].

## Context

[[ADR-012]] froze the entire [[Work Order Driver]] junction once `WorkOrder.status` advanced to `in_progress`. Any change required the full migration path from [[ADR-010 — Work Order Migration on Failure]] (close current WO as `closed_partial`, create a new WO with new assignment).

That rule fits main-driver and vehicle changes well — those are the trip identity. It is too heavy for **helpers**:

- A helper goes home sick at lunch; another joins for the afternoon. Same vehicle, same main driver, same route. Migration would close the WO mid-trip and create a new one purely to swap a loader.
- A second helper is added at a customer site to assist with heavy unloading.
- A helper's shift ends mid-trip and they are released; remaining stops continue with main only.

Each of these is a staffing nudge, not a trip identity change. The migration machinery (new WO id, new Route, history rewrite, status cascade) buys nothing here and complicates per-trip cost roll-up.

The cost model also benefits from finer granularity: a helper present for 2 hours of a 10-hour trip should bear a 2-hour duration share, not the full 10. Today there is nowhere to record per-helper duration.

## Decision

**The junction freeze in [[ADR-012]] applies only to `role = main` rows. `role = helper` rows are mutable throughout the WO lifecycle. Per-row time bounds are recorded so cost allocation can use them.**

### Schema

[[Work Order Driver]] gains time-bound fields:

| Field | Notes |
|---|---|
| `added_at` | Timestamp the row was created. Defaults to WO creation time for rows added during planning; set to current time for mid-trip additions. |
| `removed_at` | NULL while the helper is still on the trip. Set when the helper is released. Removed helpers stay in the table for cost and audit. |

### Mutation Rules

| Action | When `WO.status = planned` | When `WO.status = in_progress` | When `WO.status = completed` |
|---|---|---|---|
| Add `helper` | Allowed | **Allowed** (sets `added_at = now`) | Disallowed |
| Remove `helper` | Allowed (DELETE row) | **Allowed** (sets `removed_at = now`, row stays) | Disallowed |
| Change `helper` driver | Allowed (DELETE + INSERT) | **Allowed** (set `removed_at` on outgoing, INSERT new with `added_at`) | Disallowed |
| Add / change `main` | Allowed | Disallowed → use migration | Disallowed |
| Change vehicle | Allowed | Disallowed → use migration | Disallowed |

Once `removed_at` is set, the row is immutable.

### Cost Allocation Implication

Per-helper duration is `min(removed_at, work_ended_at) − max(added_at, driving_started_at)`, clamped to non-negative. The duration share for that helper's salary contribution becomes:

```
helper_duration_seconds = max(0,
    min(removed_at OR work_ended_at, work_ended_at)
  - max(added_at,                    driving_started_at)
)

helper_salary_share = DriverSalary.daily_share(wo.date)
                    × (helper_duration_seconds / standard_working_seconds_per_day)
```

The `main` driver continues to use the full `wo.duration_seconds = work_ended_at − driving_started_at` per [[ADR-019 — Driver Salary Allocation by Duration Share]] — main is by definition present for the full trip.

For helpers added during planning who stayed the entire trip, `added_at <= driving_started_at` and `removed_at IS NULL`, so the formula collapses to the full WO duration — same answer as the main case.

### Per-trip and Per-km Rates

Per-trip rates ([[Driver Rate]]) for a partial-trip helper are paid in full (per-trip is per-trip, not per-time). Operators that want pro-rated per-trip can express it as a per-km rate or use the manual override path.

Per-km rates use `wo.total_distance_actual_km` regardless of partial presence. If a helper is needed only for the unloading half of a long-haul trip, the operator should encode that in the helper's `Driver Rate` rather than in the WorkOrderDriver row — different pay basis, not a time fraction.

### What this does *not* change

- Main driver swap mid-trip still goes through migration ([[ADR-010]]).
- Vehicle change mid-trip still goes through migration.
- License soft-block on `main` ([[ADR-026 — Vehicle Type License Requirement and Driver Filter]]) is unchanged.
- The "exactly one `main` per WO" constraint is unchanged.

### Promotion of helper to main

If the main driver becomes incapacitated mid-trip and a helper holds the appropriate licence, the path is **migration**, not in-place promotion: close the current WO as `closed_partial`, create a new WO with the helper as `main` (and any remaining helpers carried over). This keeps main-change semantics consistent — main change always rebuilds trip identity. The existing helpers transition to the new WO via the same flow (migration creates fresh junction rows).

## Consequences

- **Good:** Common operational events (helper sick, helper join late, helper release early) stop forcing migration.
- **Good:** Per-helper duration share matches reality — short helper presence costs less than full-day presence.
- **Good:** Cost roll-up to OrderLine is unchanged — the per-stop split in [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]] still works because `WO.driver_cost` totals across helpers regardless of duration.
- **Good:** Audit trail — `added_at` / `removed_at` document staffing changes per helper.
- **Tradeoff:** The junction is no longer "snapshot at WO start" — queries that summarise helper count must filter `removed_at IS NULL` (live) vs. include all (historical).
- **Tradeoff:** Per-helper duration math adds a small computation per row. Acceptable.
- **Tradeoff:** The "main change is heavy, helper change is light" asymmetry is now an explicit design rule that the UI must reflect.

## Related

- [[Work Order]]
- [[Work Order Driver]]
- [[Driver]]
- [[Driver Rate]]
- [[Driver Salary]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — Driver × Work Order]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
