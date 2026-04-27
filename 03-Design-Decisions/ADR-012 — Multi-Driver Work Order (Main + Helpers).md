# ADR-012 — Multi-Driver Work Order (Main + Helpers)

#decision #module/work-order #module/driver-mgmt #status/active

## Status

Accepted

## Context

The original design held the driver assignment on [[Work Order]] as a single foreign key: `Work Order.driver_id`. This assumed exactly one paid person per Work Order.

In practice, transport operations often run with one main driver plus one or more **helpers**: loaders, second drivers, assistants. Helpers are paid for their work, frequently at different rates than the main driver, and sometimes through the same compensation scheme (per-trip and per-km). The single-FK design could not represent this without:

- Folding helper cost into vehicle/route overhead (loses traceability), or
- Maintaining a parallel "helper" table outside the Work Order model (creates two cost paths to reconcile).

Additionally, the same person may serve as `main` on one Work Order and `helper` on another — pay rates need to differ by role even for the same Driver.

## Decision

**A Work Order has many Drivers, captured by a junction table [[Work Order Driver]] that carries a `role` field of `main` or `helper`.**

```
Work Order  1──N  WorkOrderDriver  N──1  Driver
                  ↑
                  role enum(main, helper)
```

### Rules

1. **Exactly one `main` per Work Order.** Enforced by DB constraint or application invariant.
2. **Zero or more `helpers` per Work Order.** Unbounded.
3. **`Work Order.driver_id` is removed.** All driver references go through the junction.
4. **License soft-block applies only to `main` assignments.** Helpers may not hold a driving license.
5. **Junction rows are mutable while `WorkOrder.status = planned`.** Frozen on `in_progress`.
6. **Mid-trip changes go through migration**, not junction edits — see [[ADR-010 — Work Order Migration on Failure]].

### Cost Treatment

[[Driver Rate]] is keyed by `(driver_id, role, pay_basis, scope)`. Per-trip and per-km rates differ by role. Monthly base salary is role-agnostic (see [[Integration — Driver × Cost Ledger]]).

```
work_order.driver_cost = Σ over WorkOrderDriver(wo):
    monthly_share(driver, wo.date)
  + per_trip(driver, role, route, customer)
  + per_km  (driver, role, route, customer) × wo.total_distance_actual_km
```

### Mobile App Visibility

Both main and helpers see the Work Order in their mobile app feed. Mutating actions (status updates, evidence, odometer) are restricted to `main`. Submitting [[Fuel Log]] is open to either role.

## Consequences

- **Good:** Helper compensation lives natively in the cost model, traceable per WO and per driver.
- **Good:** A driver can be main on one WO and helper on another with correct rate resolution.
- **Good:** Adding/removing helpers during planning is cheap — no schema migration.
- **Good:** Aligns with [[ADR-008 — WorkOrderStop as Movable Join]] philosophy: relationships move via junctions, not by rewriting parent rows.
- **Tradeoff:** Cost queries now sum across N junction rows instead of reading a single FK. Acceptable.
- **Tradeoff:** Old code paths reading `Work Order.driver_id` must be migrated to read the junction.

## Related

- [[Work Order]]
- [[Work Order Driver]]
- [[Driver]]
- [[Driver Rate]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — Driver × Work Order]]
