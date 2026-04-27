# Fuel Log

#concept #module/driver-mgmt #module/vehicle #module/cost-ledger #status/active

> A record of a fuel refill against a vehicle. Operational history and source of fuel cost events.

## Definition

A Fuel Log captures a single refilling event: which vehicle, by which driver, when, how much fuel, how much it cost. It is recorded primarily through the driver mobile app but may also be entered by controllers via the web app.

Fuel Logs are tied to **vehicles only**, not Work Orders. A driver may refuel between trips, after hours, or partway through a multi-WO day; binding fuel to a specific WO would force artificial allocation and create disputes. Fuel cost is a vehicle cost — see [[Integration — Fuel Log × Cost Ledger]].

## Key Fields

| Field | Type | Description |
|---|---|---|
| `fuel_log_id` | string | Unique identifier |
| `vehicle_id` | string | Vehicle that received the fuel |
| `driver_id` | string | Driver who recorded the entry |
| `refilled_at` | datetime | When the refill happened |
| `liters` | decimal | Volume |
| `amount_thb` | decimal | Total paid |
| `station_name` | string | Free-text station / location |
| `photo_url` | string | Receipt photo, optional |
| `payment_method` | enum | `cash` / `company_card` / `voucher` / `other` — informational only |
| `created_at` | datetime | |

## Cost Event Trigger

Each Fuel Log inserts a [[Cost Event]] keyed to the vehicle:

```
CostEvent(
  entity_type = vehicle,
  entity_id   = fuel_log.vehicle_id,
  cost_type   = fuel,
  amount      = fuel_log.amount_thb,
  event_date  = fuel_log.refilled_at,
  source_ref  = fuel_log.fuel_log_id
)
```

The Cost Event flows through the vehicle cost pool and contributes to the vehicle's variable cost rate over time. It does **not** charge a specific Work Order. See [[Integration — Fuel Log × Cost Ledger]].

## Lifecycle

```
Driver opens mobile app → "Log fuel"
  Enter: vehicle (defaulted from current WO if any), liters, amount, station, photo
  Submit
  → INSERT FuelLog
  → INSERT CostEvent (fuel)
```

Edits and deletions of a Fuel Log must produce a corresponding reversal Cost Event — Cost Events are append-only per [[ADR-001 — Close and Replace, Never Mutate]].

## Related

- [[Driver]] — submitter
- [[Cost Event]]
- [[Integration — Fuel Log × Cost Ledger]]
- [[ADR-001 — Close and Replace, Never Mutate]]
