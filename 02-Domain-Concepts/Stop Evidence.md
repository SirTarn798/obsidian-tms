# Stop Evidence

#concept #module/work-order #status/active

> Operational proof attached to a [[Work Order Stop]]. Photos, signatures, documents captured at time of delivery.

## Definition

Stop Evidence records what happened at a stop — proof of delivery photos, customer signatures, damage documentation, driver notes. Multiple evidence records per stop. Separate from [[Work Order Stop]] to avoid bloating the join table with binary/media data.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `stop_evidence_id` | string | Unique identifier |
| `work_order_stop_id` | string | Parent [[Work Order Stop]] |
| `type` | enum | `photo` / `signature` / `document` / `note` |
| `url` | string | Storage URL for media files |
| `captured_at` | datetime | When evidence was captured (device time) |
| `uploaded_at` | datetime | When synced to server |
| `uploaded_by` | string | Driver/user who uploaded |
| `notes` | string | Context or description |

## Types

| Type | Use case |
|---|---|
| `photo` | Proof of delivery, condition of goods, site conditions |
| `signature` | Customer signature on receipt |
| `document` | Delivery note scan, customs doc, waybill |
| `note` | Driver text note (no media) |

## Relation to Work Order Stop

```
WorkOrderStop 1 ──→ N StopEvidence
```

Evidence is appended throughout the stop execution. All evidence for a stop is preserved even after stop is completed or migrated.

## Billing Relevance

On disputed invoices, StopEvidence provides proof that a stop was executed. Links to `work_order_stop_id`, which links back through [[Order Location]] → [[Order Line]] → [[Order]] for full traceability.

## Related

- [[Work Order Stop]] — parent
- [[Work Order]] — grandparent
