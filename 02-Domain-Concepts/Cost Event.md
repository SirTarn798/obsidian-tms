# Cost Event

#concept #module/cost-ledger

> A one-time cost that occurred at a specific point in time.

## Definition

A Cost Event records something that happened once — a purchase, a repair, a bonus. Unlike a [[Cost Rule]], it has no duration or frequency. It exists at a single point in time.

## Fields

|Field|Type|Description|
|---|---|---|
|`id`|int|Unique identifier|
|`entity_type`|enum|`vehicle`, `driver`, `user`, `facility` (extensible)|
|`entity_id`|string|Reference to the specific entity|
|`cost_type`|string|Descriptive label: `purchase`, `repair`, `fuel`, `toll`, `bonus`, `disposal_sale`, etc.|
|`cost_classification`|enum|`fixed_time`, `variable_usage`, `one_off` — drives view selection per [[ADR-014 — Cost Classification]]|
|`amount`|decimal|May be negative for refunds, payouts, disposals, reversals — see [[ADR-018 — Negative Amounts and Disposal]]|
|`occurred_on`|date|When it happened|
|`work_order_id`|string?|Optional. When set, this Event is a per-WO incidental — see [[ADR-015 — Per-WO Cost Events]]|
|`source_type`|enum|`manual`, `mobile_entry`, `fuel_log`, `maintenance`, `repair`, `reversal`, `correction`|
|`source_ref`|string?|Reference to originating record (FuelLog id, original CostEvent id for reversals, etc.)|
|`metadata`|json?|Flexible extra data (repair description, invoice ref, etc.)|

## Examples

- Vehicle purchase: 1,000,000 on 2025-05-05 (also creates a depreciation [[Cost Rule]], see [[ADR-004 — Vehicle Purchase Depreciation Spread]])
- Engine repair: 25,000 on 2025-03-12
- Employee signing bonus: 20,000 on 2025-01-15
- Vehicle registration renewal: 5,000 on 2025-07-01
- Tire replacement: 40,000 on 2025-04-20 (see [[Vehicle Maintenance — Overview]])

## Lifecycle

**Create** — record the event when it happens. Straightforward.

**Correct** — Cost Events are append-only per [[ADR-017 — Append-Only CostEvents]]. To correct an erroneous event:

1. Insert a reversing event (`amount` opposite sign, `source_type=reversal`, `source_ref` = original event id)
2. Insert a corrected event (`source_type=correction`, `source_ref` = original event id)

`SUM(amount)` across all three rows reconciles automatically. No mutation, no soft-delete on the Event itself. The originating source record (e.g. [[Fuel Log]]) may be soft-deleted as part of the same correction flow.

**Negative amounts** — refunds, payouts, disposal sales, and reversals all use negative `amount`. The ledger is net-cost: positive = money out, negative = money in. See [[ADR-018 — Negative Amounts and Disposal]].

## How Events Feed Into Cost Calculation

**Per-WO incidentals** (`work_order_id` set, `cost_classification=one_off`): Sum directly into `WorkOrder.events_cost`.

**Variable usage** (`cost_classification=variable_usage`, e.g. fuel, repair, tire): Feed into [[Cost Per Km]] buckets per [[ADR-016 — Cost Per Km Buckets]]. Smoothed across a lookback window and distributed over km driven, producing a stable per-km rate per bucket.

**One-off non-WO** (`cost_classification=one_off`, no `work_order_id`, e.g. vehicle purchase, disposal): Captured in cash-flow / accounting export by `occurred_on`. Excluded from per-km smoothing.

## Related

- [[Cost Rule]] — for ongoing costs
- [[Cost Per Km]] — how variable_usage events become per-km rates
- [[Work Order]] — per-WO incidentals via `work_order_id`
- [[ADR-014 — Cost Classification]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-017 — Append-Only CostEvents]]
- [[ADR-018 — Negative Amounts and Disposal]]