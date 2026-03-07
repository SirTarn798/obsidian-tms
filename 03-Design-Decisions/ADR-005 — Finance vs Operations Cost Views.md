# ADR-005 — Finance vs Operations Cost Views

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

The system serves two audiences with conflicting needs:

**Finance** needs to reconcile with real cash flow. If 40,000 left the bank account for repairs in March, the March report must show 40,000. Anything else creates discrepancies with accounting, tax filings, and bank statements.

**Operations** needs stable, comparable delivery costs. A single expensive repair shouldn't make every delivery that month look unprofitable and next month look artificially cheap.

## Decision

**These are two separate views of the same underlying data. Never merge them.**

### Finance View (Layer 1 — Ledger)

- Monthly report sums actual [[Cost Rule]] costs and [[Cost Event]] amounts for the period
- No smoothing, no allocation
- Matches bank account / invoices
- Used for: P&L, budgeting, tax, management accounting

### Operations View (Layer 3 — Delivery Costing)

- Fixed costs: daily depreciation + insurance + other rules, allocated per delivery proportional to time/distance
- Variable costs: [[Cost Per Km]] rate × delivery distance
- Cost/km is a **smoothed rate** calculated over a lookback window (e.g., 6 months of maintenance history ÷ total km driven)
- Used for: delivery pricing, route profitability, job acceptance decisions

### The Gap Is Information

The sum of all delivery cost allocations in a month will be _less than_ the finance view total, because vehicles have idle time. This gap = idle cost = underutilization. It's a useful metric, not a bug.

```
Finance total for May:     285,000
Sum of delivery allocations: 241,000
Idle/unallocated cost:       44,000 (15.4% — fleet utilization signal)
```

### Management Dashboard (Optional Hybrid)

For trend dashboards where spikes create noise, show:

- Primary: actual cost (finance view)
- Secondary: rolling average or normalized cost (clearly labeled as estimate)

Never replace the actual number — only supplement it.

## Consequences

- **Good:** Finance is always reconcilable with bank/accounting
- **Good:** Operations gets stable, comparable delivery costs
- **Good:** The gap between views provides fleet utilization insight
- **Tradeoff:** Two cost numbers for the "same thing" — requires clear labeling in UI
- **Tradeoff:** Users need to understand which view to use when — documentation/training needed