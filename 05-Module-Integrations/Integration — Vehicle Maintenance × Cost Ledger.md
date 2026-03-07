# Integration — Vehicle Maintenance × Cost Ledger

#integration #module/vehicle-maintenance #module/cost-ledger #status/active

> How vehicle maintenance events and contracts flow into the cost ledger.

## Data Flow

The [[Vehicle Maintenance — Overview]] module is a **producer** of cost data. The [[Cost Ledger — Overview]] is the **consumer**.

### Maintenance Contracts → Cost Rules

When a vehicle has a scheduled maintenance contract (e.g., 5,000/month for regular servicing):

- Creates a [[Cost Rule]] with `cost_type = "maintenance_contract"`, `frequency = monthly`
- If the contract is renegotiated: close old rule, create new one ([[ADR-001 — Close and Replace, Never Mutate]])

### Repairs / Ad-hoc Maintenance → Cost Events

When a vehicle gets repaired or has unscheduled work:

- Creates a [[Cost Event]] with `cost_type = "repair"` or `"maintenance"`
- These events feed into the [[Cost Per Km]] rate calculation

### Tire Replacements → Cost Events (+ optional depreciation rule)

Tires are interesting — they're a significant cost that wears down with usage:

- Record the purchase as a [[Cost Event]]
- Optionally create a depreciation [[Cost Rule]] similar to [[ADR-004 — Vehicle Purchase Depreciation Spread]] but with expected tire life in km rather than time
- See [[Vehicle Maintenance — Overview]] for tire tracking details

## What the Maintenance Module Must Provide

1. Every maintenance action creates a cost record (event or rule) — no "invisible" costs
2. Each record must include `vehicle_id` so costs are attributed correctly
3. Odometer readings at maintenance time (needed for [[Cost Per Km]] calculation)

## Related

- [[Vehicle Maintenance — Overview]]
- [[Cost Ledger — Overview]]
- [[Cost Per Km]]
- [[Integration — Cost Ledger × Delivery Costing]]