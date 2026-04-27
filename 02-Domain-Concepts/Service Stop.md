# Service Stop

#concept #module/order-mgmt #status/active

> One template stop within a [[Service]]. Defines stop type and location — copied to [[Order Location]] at order time.

## Definition

Service Stop is the template record for a stop within a [[Service]]. When an [[Order Line]] is created from a Service, each Service Stop is copied as a new [[Order Location]] on that OrderLine. The copy is adjustable before the order is confirmed.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `service_stop_id` | string | Unique identifier |
| `service_id` | string | Parent Service |
| `customer_location_id` | string | Reference to [[Customer Location]] master |
| `stop_type` | enum | `pickup` or `drop` |
| `sequence` | integer | Order within the Service's route |
| `notes` | string | Operational defaults (e.g. "call ahead 30 min") |

## Copy Behavior at Order Time

When OrderLine is created from this Service:

```
ServiceStop → OrderLocation (copy)
  customer_location_id  → (FK kept for reference)
  stop_type            → copied
  sequence             → copied as initial sequence
  address / lat / lng  → copied from [[Customer Location]] master at that moment
  notes                → copied as default, editable
```

After copy, [[Order Location]] is independent. Changes to Service Stop do not affect existing OrderLocations.

## Related

- [[Service]] — parent
- [[Customer Location]] — master location referenced
- [[Order Location]] — the snapshot copy created at order time
