# Customer Location

#concept #module/order-mgmt #status/active

> Master record of a physical location the fleet operates to/from. Mutable source of truth for address and geo data.

## Definition

Customer Location is the reusable master — a customer's warehouse, a supplier depot, a terminal. It is the template; [[Order Location]] is the immutable snapshot copy used in actual orders. Customer Location can be updated freely without affecting order history.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `customer_location_id` | string | Unique identifier |
| `customer_id` | string | Owning customer (nullable if shared facility) |
| `name` | string | Display name |
| `address` | string | Full address |
| `lat` | decimal | |
| `lng` | decimal | |
| `contact_name` | string | Site contact |
| `contact_phone` | string | |
| `notes` | string | Operational notes (gate codes, restricted hours, etc.) |

## Relation to Order Location

Each [[Order Location]] copies fields from Customer Location at order creation time. Updates to this master record affect only future orders, never existing ones.

See [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]].

## Related

- [[Order Location]] — snapshot consumers
- [[Service Stop]] — template stops reference this master
- [[Route]] — route_stops reference Customer Location IDs for display
