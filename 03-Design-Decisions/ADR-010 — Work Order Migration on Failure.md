# ADR-010 — Work Order Migration on Failure

#decision #module/work-order #status/active

## Status

Accepted

## Context

Mid-trip failures happen: vehicle breakdown, driver medical incident, accident. Some stops may already be completed when the failure occurs. The remaining stops must be reassigned to a new vehicle and driver without losing the record of what was already done, and without affecting Order revenue.

## Decision

**Failed Work Orders are preserved. Unfinished stops migrate to a new Work Order.**

Following [[ADR-001 — Close and Replace, Never Mutate]]: do not modify the failed Work Order. Close it, create a new one.

### Migrate Operation

```
Given: WO-A with stops [OL1=done, OL2=done, OL3=failed, OL4=planned]
       Failure occurs after OL2.

Step 1 — Close WO-A:
  WO-A.status = closed_partial
  WO Stop (OL3): status = migrated
  WO Stop (OL4): status = migrated
  Order Location OL3: status = migrated
  Order Location OL4: status = migrated

Step 2 — Create WO-B:
  WO-B.vehicle_id = new vehicle
  WO-B.driver_id = new driver
  WO-B.migrated_from_work_order_id = WO-A
  WO Stop (OL3): new row, work_order_id=WO-B, status=planned
  WO Stop (OL4): new row, work_order_id=WO-B, status=planned
  Order Location OL3: status = planned
  Order Location OL4: status = planned
  New Route computed for WO-B
```

### Cost Attribution

- WO-A.cost = cost of the trip up to point of failure (distance driven, time, fixed share)
- WO-B.cost = cost of completing the remaining stops
- Total route cost = WO-A.cost + WO-B.cost (two separate work orders)

### Revenue Attribution (Ops View)

- WO-A.display_revenue = allocated shares of OL1 + OL2 (done drop stops only)
- WO-B.display_revenue = allocated shares of OL3 + OL4 (when completed)

### Order Revenue: Untouched

Order revenue and billable_revenue are computed on the Order, not on Work Orders. Migration does not affect invoicing.

## Consequences

- **Good:** Full audit trail — WO-A preserved with its completed stops
- **Good:** Order revenue unaffected by operational failures
- **Good:** Follows same close-and-replace principle as rest of system
- **Tradeoff:** A single customer trip may produce 2+ Work Orders in cost reports — UI should surface `migrated_from` chain
- **Tradeoff:** Cost of failure (two trips for one delivery) is visible and real — this is correct behavior
