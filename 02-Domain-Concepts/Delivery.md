# Delivery

#concept #module/order-mgmt #module/cost-ledger

> A unit of work — a vehicle transporting goods from origin to destination. The fundamental unit for cost allocation and revenue attribution.

## Definition

A Delivery is what connects the cost world to the revenue world. Each delivery has costs (fuel, driver time, vehicle wear) and revenue (what the customer pays). The difference is your margin on that delivery.

## Key Fields

|Field|Type|Description|
|---|---|---|
|`delivery_id`|string|Unique identifier|
|`vehicle_id`|string|Which vehicle performed this delivery|
|`driver_id`|string|Which employee drove (links to [[Employee Management — Overview]])|
|`distance_km`|decimal|Total distance traveled|
|`started_at`|datetime|When the delivery began|
|`completed_at`|datetime|When the delivery was completed|
|`order_id`|string|Link to the [[Order Management — Overview]]|

## Delivery Cost Breakdown

See [[Integration — Cost Ledger × Delivery Costing]] for full details.

```
Total delivery cost =
  Fixed cost share (vehicle depreciation + insurance for that time period)
  + Variable cost (cost_per_km × distance)
  + Driver cost (driver's daily/hourly rate × time)
  + Fuel cost (actual or estimated)
```

## Related

- [[Integration — Cost Ledger × Delivery Costing]]
- [[Cost Per Km]]
- [[VRP — Overview]] — route optimization decides which vehicle does which delivery
- [[Order Management — Overview]] — the order that created this delivery
- [[Work Order Income — Overview]] — revenue side of this delivery