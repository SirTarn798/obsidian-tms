# ADR-014 — Cost Classification

#decision #module/cost-ledger #status/active

## Status

Accepted

## Context

The Cost Ledger uses two primitives — [[Cost Rule]] and [[Cost Event]] — keyed by `entity_type` and `cost_type`. `cost_type` is a free string (`salary`, `maintenance`, `repair`, `tire_replacement`, `depreciation`, `insurance`, `toll`, etc.).

Several views drive their logic off `cost_type` lists:

- [[Cost Per Km]] includes `["repair", "maintenance", "tire_replacement"]`
- [[Monthly Operational Cost]] aggregates rules and events
- Per-WO cost allocation distinguishes vehicle/driver from user/facility

Hardcoding `cost_type` lists in queries is fragile. A new cost type ("electric_charge") is silently miscategorized. A typo silently changes the calculation. The classification logic lives in code, not data.

## Decision

**Add a `cost_classification` field to both [[Cost Rule]] and [[Cost Event]]. Drive view selection off this field, not off `cost_type`.**

| Classification | Behavior | Examples |
|---|---|---|
| `fixed_time` | Cost accrues with calendar time, regardless of usage | Salary, depreciation, insurance, registration, office rent, maintenance contract |
| `variable_usage` | Cost scales with usage (km, hours) | Fuel, repair events, tire replacement |
| `one_off` | Single point-in-time obligation, not smoothed | Vehicle purchase, toll, parking fee, bonus, disposal sale |

`cost_type` remains a descriptive label (free string). `cost_classification` is the controlled enum that drives logic.

### View selection by classification

| View | Includes |
|---|---|
| [[Cost Per Km]] | `variable_usage` events on a vehicle, smoothed over lookback window |
| [[Monthly Operational Cost]] | `fixed_time` rules with `entity_type=user` or `entity_type=facility` |
| Per-WO fixed cost share | `fixed_time` rules with `entity_type=vehicle` |
| Per-WO variable cost | `cost_per_km` (derived from `variable_usage`) × WO actual distance |
| Per-WO incidentals | `one_off` events with `work_order_id` set ([[ADR-015 — Per-WO Cost Events]]) |
| Cash flow / accounting export | All entries by `occurred_on` / `effective_from` (classification-agnostic) |

### Migration of existing cost types

| cost_type | classification | Notes |
|---|---|---|
| `salary` | `fixed_time` | Driver and office staff |
| `depreciation` | `fixed_time` | Vehicle daily depreciation |
| `insurance` | `fixed_time` | Vehicle and other |
| `registration` | `fixed_time` | Annual or monthly |
| `office_rent` | `fixed_time` | Facility |
| `maintenance_contract` | `fixed_time` | Scheduled servicing |
| `fuel` | `variable_usage` | From Fuel Log events |
| `repair` | `variable_usage` | Ad-hoc repair events |
| `tire_replacement` | `variable_usage` | Tire purchase events |
| `parts` | `variable_usage` | Maintenance parts events |
| `purchase` | `one_off` | Vehicle purchase event |
| `toll` | `one_off` | Per-WO incidental |
| `parking` | `one_off` | Per-WO incidental |
| `port_pass` | `one_off` | Per-WO incidental |
| `bonus` | `one_off` | Driver or staff |
| `disposal_sale` | `one_off` | Negative amount |

## Consequences

- **Good:** Adding a new `cost_type` doesn't risk silent miscategorization — operator picks classification at creation
- **Good:** View logic is data-driven, not hardcoded
- **Good:** Audit-friendly — classification visible at the row level
- **Tradeoff:** One more required field on every Rule and Event
- **Tradeoff:** Existing views must map their hardcoded `cost_type` lists to classifications during the design rollout

## Related

- [[Cost Rule]]
- [[Cost Event]]
- [[Cost Per Km]]
- [[Monthly Operational Cost]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-016 — Cost Per Km Buckets]]
