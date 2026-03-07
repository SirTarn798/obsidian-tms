# Cost Rule

#concept #module/cost-ledger

> An ongoing cost obligation tied to an entity, with a defined time range and proration strategy.

## Definition

A Cost Rule represents "from date X to date Y (or ongoing), this entity costs Z per period." It is the primary building block of the [[Cost Ledger — Overview]].

## Fields

|Field|Type|Description|
|---|---|---|
|`id`|int|Unique identifier|
|`entity_type`|enum|`employee`, `vehicle`, (extensible)|
|`entity_id`|string|Reference to the specific entity|
|`cost_type`|string|`salary`, `maintenance`, `insurance`, `depreciation`, etc.|
|`amount`|decimal|Per-frequency amount|
|`frequency`|enum|`daily`, `monthly`, `yearly`|
|`proration_strategy`|enum|`daily_prorate`, `full_period`, `none`|
|`effective_from`|date|When this rule starts|
|`effective_until`|date?|When this rule ends (null = still active)|
|`metadata`|json?|Flexible extra data|

## Lifecycle

See [[ADR-001 — Close and Replace, Never Mutate]] for the core principle.

**Create** — a new ongoing cost begins:

- Employee hired → salary rule
- Vehicle purchased → depreciation rule (see [[ADR-004 — Vehicle Purchase Depreciation Spread]])
- Insurance contract signed → insurance rule

**Close (disable)** — the cost stops:

- Employee resigns → close salary rule on last day (auto-prorates)
- Vehicle sold → close all vehicle rules
- Contract ends → close on end date

**Replace (edit)** — the cost changes:

- Salary raise → close old rule + create new rule
- Insurance premium changes → close old + create new
- Never mutate the existing rule

**Re-enable** — cost resumes after a gap:

- Rehired employee → create new rule (don't reopen old one)
- Vehicle returns from long-term repair → create new maintenance rule

## One Entity, Many Rules

A single entity typically has multiple concurrent rules:

- Vehicle: depreciation + maintenance + insurance + registration
- Employee: salary + social security contribution + benefits

Each is independent — they can start/end/change at different times.

## Related

- [[Cost Event]] — for one-time costs
- [[Cost Ledger — Overview]] — system overview
- [[ADR-003 — Month-by-Month Proration]] — how rules are computed for date ranges