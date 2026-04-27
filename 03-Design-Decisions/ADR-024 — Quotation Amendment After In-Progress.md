# ADR-024 — Quotation Amendment After In-Progress

#decision #module/order-mgmt #status/active

## Status

Accepted. Closes the open question on the Order Management overview about post-`in_progress` amendments.

## Context

After an [[Order]] reaches `in_progress` (one or more [[Work Order Stop]]s exist, possibly executed), customers still ask for changes:

- "Add a drop at our other warehouse on the way."
- "Pick up two extra crates from supplier B."
- "Cancel the third drop, we sorted it ourselves."

The previous design left this open. Two failure modes were possible:

1. **Mutate the existing Order** — edit OrderLines and OrderLocations in place. Breaks the snapshot invariant: revenue and cost on already-executed stops would change retroactively. Audit trail destroyed.
2. **Force a brand-new Order** — clean for the data model but operationally awkward. The customer thinks of one job; splitting it across two Order ids confuses billing, dispatch, and customer comms.

A middle path is needed: extend the existing Order without rewriting its history.

## Decision

**Order amendment after `in_progress` is append-only. New OrderLines and OrderLocations may be added; existing ones with any associated WorkOrderStop are immutable. Cancellation is a status transition, not a delete.**

### Permitted operations on an `in_progress` Order

| Operation | Allowed? | Notes |
|---|---|---|
| Add new OrderLine | ✓ | Snapshots a fresh PriceRule + AllocationRule from the current Quotation Service. Adds revenue. |
| Add new OrderLocation to an existing OrderLine | ✓ | Allowed only on lines that have not yet been fully executed. Re-allocates revenue per the line's AllocationRule. |
| Edit field on an OrderLine | ✗ | Snapshot is locked. Cancel + re-add if needed. |
| Edit field on an OrderLocation that has any WorkOrderStop | ✗ | Snapshot is locked. |
| Edit field on an OrderLocation with **no** WorkOrderStop yet | ✓ | Pre-VRP-touch only. Once a stop is created, it locks. |
| Cancel an OrderLocation | ✓ | Status transition to `cancelled`. Any planned WorkOrderStop is cancelled with it; executed stops are immutable. |
| Cancel an OrderLine | ✓ | All its OrderLocations cancel with it. |
| Cancel the whole Order | ✓ | Order status → `cancelled`. Same cascade. |

### Lock rule, formally

> An OrderLine or OrderLocation is **locked for edit** the moment it has at least one [[Work Order Stop]] referencing it (regardless of stop status). Lock means: snapshot fields, references to PriceRule / AllocationRule, allocation parameters, and the existing OrderLocations on the line are immutable. New OrderLocations may be **appended** to a locked OrderLine; existing OrderLocations cannot be reordered, edited, or removed (cancellation is a status transition, not a removal).

Lock is a structural constraint, not a status. It does not prevent status transitions on the Location itself (`done`, `failed`, `cancelled`, `migrated`).

Sequencing of newly appended OrderLocations is set at append time and follows them. The original sequence of locked stops is preserved.

### Revenue and cost impact

- **Revenue.** Adding an OrderLine increases `Order.revenue` by the new line's revenue, computed from its own snapshot. Cancellation drops the cancelled line's contribution from `billable_revenue` per the [[ADR-011 — Partial Invoice Policy per Customer]] policy.
- **Cost.** Adding stops adds new WorkOrderStops, which contribute through the cost-back formula ([[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]). Cancellation: cancelled OrderLocations have no WorkOrderStop in `done` and contribute zero cost.
- **Allocation.** When a new OrderLocation joins an OrderLine, `OrderLocation.allocated_share` recomputes for that line per its AllocationRule. The line's snapshot is unchanged; the allocation is a derived view on the live set of OrderLocations.

### What about Quotation changes during the Order's life

The Quotation Service entries can be re-priced (close-and-replace per [[ADR-007 — PriceRule Mirrors CostRule]]). Already-confirmed Order Lines are unaffected because they carry their own snapshot. **A new OrderLine added by amendment uses whatever Quotation Service resolves at amendment time** — i.e. the current price, not a price frozen at original Order creation. This is the intended behaviour: a mid-job add-on is a fresh commitment.

### What is *not* covered

- Changing the customer of an existing Order. Not supported — create a new Order under the right customer.
- Moving an OrderLine between Orders. Not supported — same reason.
- Splitting an Order into two. Not supported.

These are reorganisations, not amendments.

## Consequences

- **Good:** Customer's notion of "one job" maps to one Order, even with mid-flight changes.
- **Good:** Audit trail is preserved — nothing already executed gets rewritten.
- **Good:** Revenue and cost both flow naturally — the cost-back formula and AllocationRule operate on the current set of children with no special cases.
- **Good:** Lock rule is a single, mechanical check (`HAS WorkOrderStop?`).
- **Tradeoff:** Operators cannot "fix a typo" on a locked OrderLine. Cancel + re-add is the path. Acceptable — the lock protects historical truth.
- **Tradeoff:** Mid-job add-ons price at current Quotation rates, which may differ from the original Order's prices. This is a feature, not a bug — but UX must surface the price clearly at amendment time.

## Related

- [[Order]]
- [[Order Line]]
- [[Order Location]]
- [[Work Order Stop]]
- [[Quotation]]
- [[Quotation Service]]
- [[Allocation Rule]]
- [[Model — Order to Work Order Chain]]
- [[ADR-007 — PriceRule Mirrors CostRule]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-011 — Partial Invoice Policy per Customer]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
