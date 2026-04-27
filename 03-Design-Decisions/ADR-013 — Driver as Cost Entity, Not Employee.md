# ADR-013 — Driver as Cost Entity, Not Employee

#decision #module/driver-mgmt #module/cost-ledger #status/active

## Status

Accepted. Supersedes the Employee Management module.

## Context

The original system modeled every system actor as an `Employee`, with a `User` row layered on top for authentication. This design forced controllers, planners, accountants, and drivers into a single HR-style table that demanded irrelevant fields (job title, department, hire formality, employment contract type, etc.) for non-HR purposes.

This is a Transport Management System, not an HR system. The cost-relevant question is narrower: **who incurs transport cost**, and what is the rate? The answer is "drivers and helpers" — not the controller, not the CEO, not the office cleaner.

Coupling auth to HR also created friction:

- A controller hired part-time required a full Employee record before any User permissions could exist.
- Removing a person from the system risked breaking either auth, cost history, or both.
- A helper who never logs in still had to be entered as an Employee to be paid.

## Decision

**Drivers are the only humans charged to per-Work-Order cost. Office staff salaries are still tracked in the [[Cost Ledger]] but flow to [[Monthly Operational Cost]] as overhead — not allocated to any individual Work Order.**

### Three concepts, separated

```
[[Driver]]   — operational record; license, contact, compensation rates
User         — auth record; username, password, role, optional driver_id link
Role         — permission set (controller, planner, driver-mobile, viewer, ...)
```

Relationships:

- A controller has a User row only. `User.driver_id IS NULL`.
- A driver who never logs in has a Driver row only. No User row.
- A driver who uses the mobile app has both, joined by `User.driver_id`.

See [[Authentication Scenarios]] for full identity flows.

### Cost ledger scope

Cost rules live in two reporting buckets, distinguished by `entity_type`:

| Bucket | Entity types | Allocated to WO? | Reported in |
|---|---|---|---|
| Transport-direct | `vehicle`, `driver` | Yes | `work_order.cost`, [[Integration — Order × Cost Ledger]] |
| Overhead | `user`, `facility` (future) | No | [[Monthly Operational Cost]] |

Both buckets share the same Cost Ledger primitives ([[Cost Rule]], [[Cost Event]]). The split is a reporting distinction, not a storage one. See [[Integration — User × Cost Ledger]] for office salary mechanics.

The deprecated `employee` entity type is replaced by `driver` (transport-direct) and `user` (overhead). No `staff` entity type exists.

### Migration

The previous Employee Management module is **deprecated** and out of scope. No data migration is performed. Drivers are entered fresh into the new module.

## Consequences

- **Good:** Driver records hold only fields relevant to transport — license, contact, rates. Onboarding a new driver does not require HR data entry.
- **Good:** Office staff onboarding is a single User INSERT. No Employee row needed.
- **Good:** Per-WO cost stays narrow and meaningful (`entity_type ∈ {vehicle, driver}`); per-route profitability is not polluted by overhead allocation.
- **Good:** Office salaries are still visible to finance via [[Monthly Operational Cost]] — nothing is hidden.
- **Good:** Helpers (paid but often without licenses, often part-time) are first-class — they get a Driver row and rate set without HR fields.
- **Good:** Auth is decoupled from cost. A driver who stops working but stays in the system as a controller closes their cost rules and gets a separate User per [[Authentication Scenarios]] (F1).
- **Tradeoff:** Two reporting buckets in the same ledger. Code that aggregates cost must filter by `entity_type` per its purpose (per-WO vs Monthly Operational).
- **Tradeoff:** The previous `Integration — Employee Management × Cost Ledger` document is replaced by [[Integration — Driver × Cost Ledger]] + [[Integration — User × Cost Ledger]].

## Alternatives Considered

### A. Keep Employee, narrow its purpose

Reduce Employee to "people who draw a salary," with Driver as a sub-record. Rejected — still mixes HR-shaped data with operational data; office staff still requires a row; cost ledger still has to discriminate by sub-type to avoid charging controllers.

### B. Polymorphic cost entity, all flows to per-WO

Allow `CostRule.entity_type ∈ {vehicle, driver, user, ...}` and allocate every entry to per-WO cost via some allocation key. Rejected — there is no honest allocation key for controller salary across trips. This was the original concern; the chosen design instead **keeps polymorphism but splits the rollup**: vehicle/driver to per-WO, user/facility to Monthly Operational Cost.

### C. Driver is a User role

Conflate Driver into the User table with `role=driver`. Rejected — a driver who never logs in has no business needing a User row, and decoupling cost from auth is the whole point.

## Related

- [[Driver]]
- [[Driver Rate]]
- [[Authentication Scenarios]]
- [[Driver Management — Overview]]
- [[Monthly Operational Cost]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — User × Cost Ledger]]
- [[ADR-005 — Finance vs Operations Cost Views]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
