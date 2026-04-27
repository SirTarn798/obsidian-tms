# ADR-015 — Per-WO Cost Events

#decision #module/cost-ledger #module/work-order #status/active

## Status

Accepted

## Context

A [[Work Order]] incurs incidental costs that are real, money-out-the-door, attributable to a specific trip — but not to a vehicle pool or driver pool. Examples: tolls, parking fees, port pass fees, customer-specific extras (forklift hire, container handling, document fees).

The original Cost Ledger had no place for these. Storing them in `route_cost` / `tms_cost` / similar tables was the original "scattered cost" problem this redesign addresses.

Charging tolls to the vehicle's [[Cost Per Km]] is wrong: a toll for one trip would smooth into the rate that prices all future trips. Charging to overhead is wrong: the cost is operationally tied to a specific WO.

## Decision

**Add an optional `work_order_id` field to [[Cost Event]]. Per-WO incidentals live in the unified ledger as Events with this field set.**

```
CostEvent(
    entity_type         = vehicle | driver,
    entity_id           = the vehicle or driver involved,
    cost_type           = toll | parking | port_pass | forklift | document_fee | ...,
    cost_classification = one_off,
    amount              = cost incurred (may be negative for refunds),
    occurred_on         = date,
    work_order_id       = WO this is attributable to,
    source_type         = manual | mobile_entry | ...,
    source_ref          = optional reference to originating record
)
```

### Per-WO `events_cost`

```
WorkOrder.events_cost = Σ CostEvent.amount WHERE work_order_id = this WO
```

This becomes a fourth term in the WO cost formula:

```
WorkOrder.cost = fixed_cost_share + variable_cost + driver_cost + events_cost
```

### Vehicle history is preserved

A toll Event still has `entity_type=vehicle, entity_id=V1` — vehicle-level queries (`SELECT * FROM CostEvent WHERE entity_id=V1`) still find it. The `work_order_id` is an additional context field, not a replacement for the entity reference.

Both query directions work:
- WO costs: `WHERE work_order_id = WO-77`
- Vehicle history: `WHERE entity_id = V1`
- Specific scope: `WHERE entity_id = V1 AND work_order_id = WO-77`

### Smoothing exclusion

Per-WO events are `cost_classification=one_off`. They are **not** included in [[Cost Per Km]] smoothing — that's only `variable_usage` events. A one-time port fee for WO-77 doesn't become part of the long-term variable rate.

### Capture paths

| Source | Trigger | Result |
|---|---|---|
| Driver mobile app | Driver logs incidental during trip | CostEvent with `source_type=mobile_entry`, `work_order_id` from current WO |
| Controller web | Manual entry post-trip | CostEvent with `source_type=manual`, `work_order_id` selected |
| Customer-specific extra | Quoted on Order, billed separately | Same shape; revenue side handled on Order |

## Consequences

- **Good:** Per-WO incidentals are first-class in the ledger, no separate table needed
- **Good:** Both per-WO and per-vehicle queries find the same event
- **Good:** Smoothing isolation — one-off costs don't pollute long-term rates
- **Good:** Mobile entry fits naturally — driver records toll, app fills `work_order_id` from current trip
- **Tradeoff:** Operators must classify the event correctly (`one_off` not `variable_usage`)
- **Tradeoff:** UI needs an "attribute to WO" picker when entering an Event during an active trip

## Related

- [[Cost Event]]
- [[Work Order]]
- [[Integration — Order × Cost Ledger]]
- [[ADR-014 — Cost Classification]]
- [[ADR-017 — Append-Only CostEvents]]
