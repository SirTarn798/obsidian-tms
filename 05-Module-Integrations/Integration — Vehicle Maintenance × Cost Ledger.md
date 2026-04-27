# Integration — Vehicle Maintenance × Cost Ledger

#integration #module/vehicle-maintenance #module/cost-ledger #status/active

> How vehicle maintenance events and contracts flow into the cost ledger.

## Data Flow

The [[Vehicle Maintenance — Overview]] module is a **producer** of cost data. The [[Cost Ledger — Overview]] is the **consumer**.

### Maintenance Contracts → Cost Rules

When a vehicle has a scheduled maintenance contract (e.g., 5,000/month for regular servicing):

- Creates a [[Cost Rule]] with `cost_type=maintenance_contract`, `cost_classification=fixed_time`, `frequency=monthly`, `proration_strategy=daily_prorate`
- If the contract is renegotiated: close old rule, create new one ([[ADR-001 — Close and Replace, Never Mutate]])

Note: a fixed-time maintenance contract contributes to the vehicle's `fixed_cost_share` per WO, **not** to [[Cost Per Km]]. Per-km maintenance economics come from `variable_usage` repair events (below).

### Repairs / Ad-hoc Maintenance → Cost Events

When a vehicle gets repaired or has unscheduled work:

- Creates a [[Cost Event]] with `cost_type=repair`, `cost_classification=variable_usage`
- These events feed the `maintenance` bucket of [[Cost Per Km]] per [[ADR-016 — Cost Per Km Buckets]]

If a repair is incurred specifically for one trip (e.g., a tire blow-out on WO-77 paid out-of-pocket on the road), it can be classified as `one_off` with `work_order_id` set — see [[ADR-015 — Per-WO Cost Events]].

### Tire Replacements → Cost Events

Tires wear with usage. The `tire` bucket of [[Cost Per Km]] smooths replacement events into a stable per-km figure:

- Record purchase as a [[Cost Event]] with `cost_type=tire_replacement`, `cost_classification=variable_usage`
- The `tire` bucket has a longer default lookback (12 months) than `maintenance` (6 months) per [[ADR-016 — Cost Per Km Buckets]]

A separate per-tire depreciation rule with `usage_prorate` strategy is **not** part of the current design — the bucket smoothing in Cost Per Km already produces a stable per-km tire rate without additional plumbing.

## What the Maintenance Module Must Provide

1. Every maintenance action creates a cost record (event or rule) — no "invisible" costs
2. Each record must include `vehicle_id` so costs are attributed correctly
3. Odometer readings at maintenance time (needed for [[Cost Per Km]] calculation)

## Related

- [[Vehicle Maintenance — Overview]]
- [[Cost Ledger — Overview]]
- [[Cost Rule]]
- [[Cost Event]]
- [[Cost Per Km]]
- [[Integration — Order × Cost Ledger]]
- [[ADR-014 — Cost Classification]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-016 — Cost Per Km Buckets]]