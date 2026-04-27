# ADR-028 — Driver and Vehicle Overlap Soft-Block

#decision #module/work-order #module/driver-mgmt #status/active

## Status

Accepted

## Context

Today nothing prevents the controller from assigning the same Driver — or the same Vehicle — to two Work Orders that overlap in time. The system silently accepts double-bookings. The cost ledger then produces nonsense (the same daily salary share allocated to two parallel WOs; the same vehicle daily depreciation share double-counted).

The check is straightforward in principle: a Driver cannot be on two WOs whose execution windows overlap, and the same applies to a Vehicle. The wrinkle is that "execution window" means different things at different lifecycle stages:

- **Pre-execution** — `WO.status = planned`. The WO has no `driving_started_at` yet. The window is the planned trip times (from [[Route]] estimates or operator-set planned start/end).
- **In-flight** — `WO.status = in_progress`. `driving_started_at` is real; `work_ended_at` is still planned/estimated.
- **Closed** — `WO.status = completed | closed_partial | cancelled`. Both timestamps are real.

A blanket hard-block is wrong: legitimate overlaps happen — a controller backing-up one WO with a fallback driver, two WOs intentionally chained back-to-back where the planned end of one equals the planned start of another, an ad-hoc late-finish that nudges into the next assignment.

A soft-block matches the existing pattern from [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]: surface the conflict, capture an explicit override with reason and audit fields.

## Decision

**Detect Driver and Vehicle overlap across Work Orders. Soft-block on conflict. Record override on the affecting [[Work Order Driver]] row (driver overlap) or on the [[Work Order]] (vehicle overlap).**

### Window Definition

For a given WO, the **effective window** used for overlap checks is:

```
window_start =
    driving_started_at  IF status IN (in_progress, completed, closed_partial)
    ELSE planned_start

window_end =
    work_ended_at       IF status = completed | closed_partial
    ELSE planned_end    (or live estimate if in_progress)
```

`planned_start` / `planned_end` come from the [[Route]] estimate plus a small slack factor configured per tenant (default zero — overlap on the minute is still an overlap).

WOs in `cancelled` status are excluded from overlap checks.

### Driver Overlap

```
on assign(driver, role, wo):
  conflicts = SELECT other.*
              FROM WorkOrder other
              JOIN WorkOrderDriver other_wod ON other_wod.work_order_id = other.work_order_id
              WHERE other_wod.driver_id = driver.driver_id
                AND other.work_order_id != wo.work_order_id
                AND other.status != 'cancelled'
                AND (other_wod.removed_at IS NULL OR other_wod.removed_at > wo.window_start)
                AND tsrange(other.window_start, other.window_end)
                    && tsrange(wo.window_start, wo.window_end)

  if conflicts non-empty:
      surface conflict list with WO ids, dates, roles
      require explicit "Proceed anyway" + non-empty reason
      record override fields on the new WorkOrderDriver row
```

The check applies regardless of `role`. A Driver who is `main` on WO-A and `helper` on WO-B at the same time still trips the check — physically one body, one trip at a time.

### Vehicle Overlap

Same shape, keyed on `vehicle_id`:

```
on assign(vehicle, wo):
  conflicts = ... SELECT other WHERE other.vehicle_id = wo.vehicle_id ...
  if conflicts non-empty: soft-block + override on the WO itself
```

Vehicle override fields live on [[Work Order]] (vehicle is a single FK on the WO, not a junction):

| Field | Notes |
|---|---|
| `vehicle_overlap_override_by_user_id` | |
| `vehicle_overlap_override_at` | |
| `vehicle_overlap_override_reason` | Free text. Required when override fires. |

### Driver Override Fields

Live on [[Work Order Driver]] (one row per assignment, possibly per-helper):

| Field | Notes |
|---|---|
| `overlap_override_by_user_id` | |
| `overlap_override_at` | |
| `overlap_override_reason` | Free text. Required when override fires. |

(These are separate from the licence override fields in [[ADR-026]] — same row can carry both, set independently.)

### Mid-trip Helper Adds (per ADR-027)

Helper rows added mid-trip per [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]] also pass through this check. The window for the helper add is `[max(now, wo.window_start), wo.window_end]`. Practical effect: if the helper is on another WO right now, conflict; otherwise, save silently.

### Reporting

Override rows are queryable for compliance:

```sql
-- Driver overlaps
SELECT * FROM WorkOrderDriver WHERE overlap_override_at IS NOT NULL ORDER BY overlap_override_at DESC;

-- Vehicle overlaps
SELECT * FROM WorkOrder WHERE vehicle_overlap_override_at IS NOT NULL ORDER BY vehicle_overlap_override_at DESC;
```

Frequency by user is the operational signal — high override rate suggests scheduling is too tight or roster is too thin.

## Consequences

- **Good:** Catches double-bookings at assignment, before they corrupt cost.
- **Good:** Same soft-block + audit pattern as licence override — controllers learn one workflow.
- **Good:** Free-text reason captures the why; reporting picks up frequency by user/vehicle/driver.
- **Good:** Cancelled WOs don't pollute the check.
- **Tradeoff:** Window for `planned` WOs depends on `planned_start` / `planned_end` being populated. If the operator skips them, the check has nothing to compare. UI must require these fields at WO creation.
- **Tradeoff:** Live-estimate `window_end` for `in_progress` WOs drifts as the trip runs late. Acceptable — the estimate is good enough for "is anyone scheduled to start before this finishes?"
- **Tradeoff:** A driver legitimately back-up-listed across two near-simultaneous WOs trips the check every time. Operators accept that as the cost of the audit trail.

## Related

- [[Work Order]]
- [[Work Order Driver]]
- [[Driver]]
- [[Route]]
- [[Integration — Driver × Work Order]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]
