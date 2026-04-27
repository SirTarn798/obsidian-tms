# Route

#concept #module/work-order #status/active

> Geometry and metrics of a [[Work Order]]'s path. Stores polyline, distance, and estimated duration only.

## Definition

Route contains the navigation data for a [[Work Order]] — the encoded polyline, total planned distance, and estimated duration. It is the geometry container. Stop sequence and timing live on [[Work Order Stop]]s.

Separate from [[Work Order]] so routing data can be versioned independently (e.g. re-computed after road changes without touching the WO).

## Key Fields

| Field | Type | Description |
|---|---|---|
| `route_id` | string | Unique identifier |
| `work_order_id` | string | Work Order this route serves |
| `polyline` | string | Encoded polyline for map display (coordinates embedded) |
| `total_distance_km` | decimal | Total planned route length |
| `estimated_duration_min` | integer | VRP-estimated total travel time |
| `computed_at` | datetime | When VRP last generated this route |

## Planned vs Actual

`total_distance_km` here is the VRP plan. Actual distance is on [[Work Order]] (`total_distance_actual_km` from odometer). Cost allocation uses actual when available, planned as fallback.

## Stop Sequence

Stop order, planned arrivals, and actual arrivals live on [[Work Order Stop]] — not here. To render a route with stops, join Work Order → Work Order Stops → Order Locations for coordinates.

## Related

- [[Work Order]] — owns this route
- [[Work Order Stop]] — stop sequence and timing (the execution counterpart)
