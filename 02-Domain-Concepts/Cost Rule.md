# Cost Rule

#concept #module/cost-ledger

> An ongoing cost obligation tied to an entity, with a defined time range and proration strategy.

## Definition

A Cost Rule represents "from date X to date Y (or ongoing), this entity costs Z per period." It is the primary building block of the [[Cost Ledger — Overview]].

## Fields

| Field                 | Type    | Description                                                |
| --------------------- | ------- | ---------------------------------------------------------- |
| `id`                  | int     | Unique identifier                                          |
| `entity_type`         | enum    | `vehicle`, `driver`, `user`, `facility` (extensible)       |
| `entity_id`           | string  | Reference to the specific entity                           |
| `cost_type`           | string  | Descriptive label: `salary`, `maintenance_contract`, `insurance`, `depreciation`, `office_rent`, etc. |
| `cost_classification` | enum    | `fixed_time`, `variable_usage`, `one_off` — drives view selection per [[ADR-014 — Cost Classification]] |
| `amount`              | decimal | Per-frequency amount                                       |
| `frequency`           | enum    | `daily`, `monthly`, `yearly`                               |
| `proration_strategy`  | enum    | `daily_prorate`, `none` — see [[ADR-003 — Month-by-Month Proration]] |
| `effective_from`      | date    | When this rule starts                                      |
| `effective_until`     | date?   | When this rule ends (null = still active)                  |
| `metadata`            | json?   | Flexible extra data                                        |

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

- Vehicle: depreciation + maintenance contract + insurance + registration
- Driver: salary (see [[Driver Salary]])
- Office staff: salary

Each is independent — they can start/end/change at different times.

## At Most One Open Rule per (entity, cost_type)

For a given `(entity_type, entity_id, cost_type)` there is at most one Rule with `effective_until = NULL` at any time. Enforced as a database constraint or application invariant. Prevents accidental double-prorating (e.g. forgetting to close the old salary rule when creating a raise).

To replace an open rule:

1. Close the existing rule (`effective_until = ...`)
2. Insert the new rule (`effective_from = ...`)

If an operator attempts to insert a second open rule, the system rejects with a clear error and a link to the existing one.

## Related

- [[Cost Event]] — for one-time costs
- [[Cost Ledger — Overview]] — system overview
- [[ADR-001 — Close and Replace, Never Mutate]] — including the at-most-one-open-rule invariant
- [[ADR-003 — Month-by-Month Proration]] — how rules are computed for date ranges
- [[ADR-014 — Cost Classification]] — the `cost_classification` field