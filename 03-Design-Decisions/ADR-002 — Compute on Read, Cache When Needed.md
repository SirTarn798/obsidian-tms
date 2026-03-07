# ADR-002 — Compute on Read, Cache When Needed

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

We considered two approaches for answering cost queries:

1. **Pre-compute daily snapshots** via cron job, then query the snapshots
2. **Store raw events/rules**, compute any date range on demand

## Decision

**Compute on read as the primary approach.** The engine takes a date range, finds overlapping rules and events, computes month-by-month, and returns results. No pre-materialized data required.

**Add caching later if needed** as a performance optimization — not as the source of truth. A materialized monthly summary table can be rebuilt from underlying rules/events at any time.

## Why Not Pre-Compute

- **Corrections are painful.** If a salary was entered wrong 3 months ago, you'd need to recompute 90+ daily snapshots. With compute-on-read, you fix the rule and the next query is automatically correct.
- **Source of truth confusion.** When pre-computed snapshots disagree with raw data (and they will, due to bugs or timing), which one is "right"? With compute-on-read, there's only one source.
- **Storage cost for little benefit.** For the entity counts we expect (hundreds of vehicles, hundreds of employees), computing a 12-month range takes milliseconds.

## When to Add Caching

If query latency becomes a problem (unlikely before thousands of entities × years of history), add a `monthly_cost_summary` table that is:

- Rebuilt nightly from rules/events
- Used as a read cache, never as source of truth
- Invalidated when rules/events are modified for affected months

## Consequences

- **Good:** Single source of truth, always consistent
- **Good:** Corrections are trivial — fix the data, queries auto-correct
- **Good:** No cron jobs to monitor, no stale data risk
- **Tradeoff:** Slightly more computation per query (acceptable at our scale)