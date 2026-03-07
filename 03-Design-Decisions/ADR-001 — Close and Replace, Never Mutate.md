# ADR-001 — Close and Replace, Never Mutate

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

When an ongoing cost changes (employee gets a raise, insurance premium changes, maintenance contract renegotiated), we need to update the system. The naive approach is to edit the existing record. But this destroys history — if someone asks "what was our cost structure in March?", we can no longer answer correctly because March's data now shows April's values.

## Decision

**Cost Rules are immutable once created.** To change an ongoing cost:

1. Close the existing rule by setting `effective_until` to the last day of the old rate
2. Create a new rule with the new amount and `effective_from` = first day of new rate

Never delete rules. Never modify `amount`, `frequency`, or `effective_from` on an existing rule.

## Examples

**Employee salary change (45k → 50k on April 1):**

```
CLOSE: rule_id=7, effective_until=2025-03-31
CREATE: entity=emp-002, amount=50000, effective_from=2025-04-01
```

**Vehicle insurance premium increase at renewal:**

```
CLOSE: rule_id=12, effective_until=2025-06-30
CREATE: entity=vehicle-001, cost_type=insurance, amount=3500, effective_from=2025-07-01
```

**Employee termination (leaves May 20):**

```
CLOSE: rule_id=7, effective_until=2025-05-20
(no new rule — the cost simply stops)
```

**Employee rehire:**

```
(don't reopen old rule — create brand new one)
CREATE: entity=emp-002, amount=55000, effective_from=2025-09-01
(gap between May 20 and Sep 1 correctly shows zero cost)
```

## Consequences

- **Good:** Full history preserved. Any past date query returns accurate results.
- **Good:** Audit trail is inherent — you can see every change by listing rules for an entity sorted by `effective_from`.
- **Good:** No complex versioning or temporal table needed — just filter by date overlap.
- **Tradeoff:** More rows over time (two rules instead of one edited rule). This is negligible for the data volumes we'll have.
- **Tradeoff:** "Current state" requires filtering for `effective_until IS NULL` instead of just reading the row. Minor query complexity.

## Applies To

All modules that feed into the cost ledger: [[Employee Management — Overview]], [[Vehicle Maintenance — Overview]], any future cost source.