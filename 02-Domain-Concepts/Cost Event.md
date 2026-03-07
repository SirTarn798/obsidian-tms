# Cost Event

#concept #module/cost-ledger

> A one-time cost that occurred at a specific point in time.

## Definition

A Cost Event records something that happened once — a purchase, a repair, a bonus. Unlike a [[Cost Rule]], it has no duration or frequency. It exists at a single point in time.

## Fields

|Field|Type|Description|
|---|---|---|
|`id`|int|Unique identifier|
|`entity_type`|enum|`employee`, `vehicle`, (extensible)|
|`entity_id`|string|Reference to the specific entity|
|`cost_type`|string|`purchase`, `repair`, `bonus`, `registration_fee`, etc.|
|`amount`|decimal|The cost amount|
|`occurred_on`|date|When it happened|
|`metadata`|json?|Flexible extra data (repair description, invoice ref, etc.)|

## Examples

- Vehicle purchase: 1,000,000 on 2025-05-05 (also creates a depreciation [[Cost Rule]], see [[ADR-004 — Vehicle Purchase Depreciation Spread]])
- Engine repair: 25,000 on 2025-03-12
- Employee signing bonus: 20,000 on 2025-01-15
- Vehicle registration renewal: 5,000 on 2025-07-01
- Tire replacement: 40,000 on 2025-04-20 (see [[Vehicle Maintenance — Overview]])

## Lifecycle

**Create** — record the event when it happens. Straightforward.

**Correct** — if entered wrong (wrong amount, wrong vehicle), either:

- Mutate directly (events are point-in-time facts, less risk than mutating rules)
- Or for full audit trail: soft-delete the wrong event + create corrected one

**Cancel** — if recorded by mistake: soft-delete (set `cancelled_at` or `is_active = false`), don't hard-delete.

## How Events Feed Into Cost Calculation

**Finance view:** Events are summed directly in the month they occurred. A 25,000 repair in March shows as 25,000 in March's report.

**Operations view (delivery costing):** Events feed into the [[Cost Per Km]] rate calculation. The 25,000 repair gets averaged across the lookback window and distributed over km driven, resulting in a stable cost/km rate.

## Related

- [[Cost Rule]] — for ongoing costs
- [[Cost Per Km]] — how maintenance events become per-km rates