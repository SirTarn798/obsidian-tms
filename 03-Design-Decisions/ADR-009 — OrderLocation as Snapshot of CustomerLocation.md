# ADR-009 — OrderLocation as Snapshot of CustomerLocation

#decision #module/order-mgmt #status/active

## Status

Accepted

## Context

[[Customer Location]] is a master record that can be updated over time — address corrections, contact changes, geo coordinate fixes. Meanwhile, [[Order Location]] must reflect what was agreed with the customer at the time of the order, regardless of later master updates.

## Decision

**Order Location copies all address/geo fields from Customer Location at order creation time.**

`order_location.customer_location_id` exists as a reference FK for lookup and reporting, but it is not the authoritative source for any field on the Order Location. All address, lat, lng, and contact fields are copied.

### Fields copied at creation

- `address`
- `lat`, `lng`
- `contact_name`, `contact_phone`
- `notes` (relevant operational notes at time of order)

Fields not copied (set at order time, not from master):
- `stop_type` — set by the order
- `time_window_*` — set by the customer's request
- `goods_description`, `weight_kg` — set by the order

### When master data changes

- Existing Order Locations: unaffected
- New Orders: use updated master data

### Tradeoff accepted

Slight data duplication. Justified by: historical accuracy of orders, audit trail integrity, and simplicity of retrieval (no join to master needed to render order history).

## Consequences

- **Good:** Order history always shows what was agreed, regardless of master changes
- **Good:** No join to Customer Location needed for historical reporting
- **Tradeoff:** Address corrections to master don't automatically fix existing orders (by design)
- **Tradeoff:** If operator wants to apply a master correction to an in-progress order, it must be done explicitly on the Order Location
