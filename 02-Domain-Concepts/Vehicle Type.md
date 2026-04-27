# Vehicle Type

#concept #module/work-order #status/active

> The vehicle class catalog. Defines licence requirements and (later) capacity limits for routing. Referenced by [[Quotation Service]] for per-vehicle-type pricing and by [[Vehicle]] for class assignment.

## Definition

Vehicle Type is the small reference catalog of operating vehicle classes — `4-wheel`, `10-wheel`, `18-wheel`, etc. Each individual [[Vehicle]] belongs to exactly one Vehicle Type. The Vehicle Type carries class-level facts: which driving licence classes can operate it, and (in v2) capacity limits for VRP filtering.

Vehicle Type is also the price dimension on [[Quotation Service]] per [[ADR-021 — QuotationService Keyed by Vehicle Type]] — same Service can be priced differently per vehicle class.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `vehicle_type_id` | string | PK |
| `name` | string | Display name (4-wheel, 10-wheel, 18-wheel, ...) |
| `code` | string | Short code for UI / reports |
| `required_license_classes` | array<string> | Driver must hold **at least one** of these to operate without a licence override. See [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]. |
| `default_fuel_per_km` | decimal | Cold-start seed for the `fuel` CPK bucket. See [[ADR-032 — Cost Per Km Cold-Start Default from Vehicle Type]]. |
| `default_maintenance_per_km` | decimal | Cold-start seed for the `maintenance` CPK bucket. |
| `default_tire_per_km` | decimal | Cold-start seed for the `tire` CPK bucket. |
| `notes` | string | Operational notes |

## Licence Matching

A driver is eligible to operate a vehicle of type `T` as `main` when:

```
driver.license_classes ∩ T.required_license_classes ≠ ∅
AND driver.license_expiry_date >= work_order.date
```

Failing either condition triggers the soft-block + free-text override flow on [[Work Order Driver]] per [[ADR-026]]. Helpers are not licence-checked.

## Pricing Dimension

[[Quotation Service]] is keyed by `(quotation, service, vehicle_type)`. Same [[Service]] for the same Customer can be offered at different prices per vehicle type. See [[ADR-021]].

## CPK Cold-Start Defaults

The `default_*_per_km` fields seed [[Cost Per Km]] computation for vehicles with no history. As soon as a vehicle accumulates any events plus km in the lookback window, its computed CPK takes over per [[ADR-032]]. Buckets are extensible — adding a new CPK bucket per [[ADR-016 — Cost Per Km Buckets]] adds a corresponding `default_<bucket>_per_km` field here.

Sub-contracted [[Vehicle]] rows do not consult these defaults — sub-contract WO cost is shaped by [[Subcontractor Rate]] and the `subcontract_payout` event per [[ADR-029]].

## Held for v2

| Field | Why deferred |
|---|---|
| `max_payload_kg` | Needed when VRP gains load-aware vehicle selection. |
| `max_volume_m3` | Same. |
| `axle_count` | Toll / road-restriction inputs. |

The current v1 list is intentionally narrow — only what is needed to make the licence rule, the QuotationService key, and the CPK cold-start fallback concrete.

## Related

- [[Vehicle]] — instances of this type
- [[Quotation Service]] — keyed in part by vehicle type
- [[Driver]] — licence classes matched against required classes
- [[Work Order]] — assigned vehicle's type drives the licence check
- [[Work Order Driver]] — holds licence override audit
- [[ADR-021 — QuotationService Keyed by Vehicle Type]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
- [[ADR-032 — Cost Per Km Cold-Start Default from Vehicle Type]]
