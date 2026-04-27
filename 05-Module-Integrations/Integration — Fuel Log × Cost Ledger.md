# Integration — Fuel Log × Cost Ledger

#integration #module/driver-mgmt #module/vehicle #module/cost-ledger #status/active

> How [[Fuel Log]] entries become [[Cost Event]]s in the [[Cost Ledger]].

## Overview

Fuel cost is tied to vehicles, not Work Orders. A driver may refuel between trips, after hours, or partway through a multi-WO day; binding fuel to a specific WO would force artificial allocation. Instead, each Fuel Log inserts a Cost Event keyed to the **vehicle**. The cost flows through the vehicle pool and contributes to the vehicle's [[Cost Per Km]] over time.

## Data Flow

```
Driver submits Fuel Log via mobile app
  → INSERT FuelLog(vehicle_id, driver_id, refilled_at, liters, amount_thb, ...)
  → INSERT CostEvent(
        entity_type         = vehicle,
        entity_id           = fuel_log.vehicle_id,
        cost_type           = fuel,
        cost_classification = variable_usage,
        amount              = fuel_log.amount_thb,
        occurred_on         = fuel_log.refilled_at,
        source_type         = fuel_log,
        source_ref          = fuel_log.fuel_log_id
     )
```

The Cost Event is append-only per [[ADR-017 — Append-Only CostEvents]]. The `variable_usage` classification routes it to the `fuel` bucket of [[Cost Per Km]] per [[ADR-016 — Cost Per Km Buckets]].

## Why Vehicle, Not Work Order

A given fuel purchase fills the tank — not a single trip. Tying a 2,000-baht refill to one specific WO would either:

- Charge the entire 2,000 to that WO (wrong — fuel powers later trips too), or
- Force liter-based proration across future WOs (complex, brittle, hard to query).

Instead, fuel events accumulate at the vehicle level. The vehicle's variable cost rate ([[Cost Per Km]]) smooths fuel and maintenance into a per-km figure that gets applied to each WO's actual distance. See [[ADR-005 — Finance vs Operations Cost Views]].

## Edits and Reversals

Per [[ADR-017 — Append-Only CostEvents]], Cost Events are append-only. Editing or deleting a Fuel Log produces reversing/correcting events:

```
Edit fuel log: amount changed from 2,000 → 1,800
  1. INSERT CostEvent(amount=-2000, source_type=reversal,   source_ref=original_event_id)
  2. INSERT CostEvent(amount= 1800, source_type=correction, source_ref=original_event_id)

Delete fuel log:
  1. INSERT CostEvent(amount=-original_amount, source_type=reversal, source_ref=original_event_id)
  2. UPDATE FuelLog SET status=deleted
```

`SUM(amount)` over all events for the vehicle reconciles automatically. Negative amounts on reversals are valid per [[ADR-018 — Negative Amounts and Disposal]].

## Cost Per Km Refresh

[[Cost Per Km]] is a smoothed variable-cost rate per vehicle, recalculated weekly/monthly:

```
cost_per_km(vehicle, as_of_date) =
   Σ CostEvent(vehicle, cost_type IN [fuel, maintenance, ...]) over rolling window
   / Σ vehicle.total_distance_actual_km over same window
```

Fuel Log Cost Events feed directly into the numerator. No special handling needed — the formula already aggregates by `entity_type=vehicle`.

## Cost Ledger API

```
post_cost_event(entity_type=vehicle, entity_id=X, cost_type=fuel, amount=Y, event_date=Z, source_ref=fuel_log_id)
list_cost_events(entity_type=vehicle, entity_id=X, from, to)
get_cost_per_km(vehicle_id, as_of_date)
```

## Mobile API

```
POST /mobile/fuel_log
  { vehicle_id, refilled_at, liters, amount_thb, station_name, photo_url, payment_method }
  → driver_id taken from JWT (cannot spoof)
  → INSERT FuelLog
  → INSERT CostEvent
  → return { fuel_log_id }
```

The driver sees their submitted fuel logs in a personal history view. Editing past entries requires controller approval (deferred).

## Related

- [[Fuel Log]]
- [[Cost Event]]
- [[Cost Per Km]]
- [[Cost Ledger — Overview]]
- [[Driver]]
- [[Integration — Vehicle Maintenance × Cost Ledger]] — sibling integration for maintenance events
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-005 — Finance vs Operations Cost Views]]
