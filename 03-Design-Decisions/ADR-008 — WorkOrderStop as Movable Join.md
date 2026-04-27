# ADR-008 — WorkOrderStop as Movable Join

#decision #module/work-order #status/active

## Status

Accepted

## Context

VRP must be able to re-plan routes freely — merging, splitting, and re-sequencing stops across Work Orders. In the original design, Order Locations were more tightly coupled to Work Orders, making re-planning messy and requiring updates to Order-layer data.

Additionally, tracking "what was promised" vs "what was executed" was unclear because the same entity held both.

## Decision

**[[Work Order Stop]] is a freely movable join table between [[Work Order]] and [[Order Location]].**

```
Work Order  1──N  Work Order Stop  N──1  Order Location
```

### Rules

1. **Order Locations never move.** They are created once per Order and stay there.
2. **Work Order Stops are ephemeral** until a stop is executed. They can be deleted and recreated freely.
3. **Re-plan = delete + insert.** Drop old WO Stop rows for affected stops, insert new rows on new/updated Work Orders. Order Locations are untouched.
4. **Executed stops are immutable.** Once `status=done`, the WO Stop row is permanent audit record. Cannot be deleted or re-assigned.
5. **Migrated stops** get `status=migrated` (terminal) on the old WO Stop, and a new WO Stop row on the new Work Order.

### "Promised route" vs "Executed route"

- Promised: the ordered sequence of Order Locations on the Order (from `order_location.sequence`)
- Executed: the Route attached to the Work Order that actually ran

These are separate. VRP may reorder stops from the promised sequence. Both are preserved.

## Consequences

- **Good:** VRP re-planning touches only WO Stop rows — Order and revenue untouched
- **Good:** Clean audit trail — done stops are immutable records
- **Good:** Migrate (mid-trip failure) follows the same pattern — terminal status + new rows
- **Tradeoff:** Profitability query requires joining through WO Stops to Order Locations to Orders
