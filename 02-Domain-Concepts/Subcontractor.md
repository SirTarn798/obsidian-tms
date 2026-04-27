# Subcontractor

#concept #module/work-order #module/cost-ledger #status/active

> A partner company that supplies vehicles and drivers for trips on behalf of the operating tenant. Standing relationship; the actual trucks and drivers they send rotate freely trip to trip.

## Definition

A Subcontractor is the standing entity behind a contracting relationship. The tenant may have contracts with several subcontractors, each able to supply one or more vehicle types. The trucks and drivers they send vary trip to trip — what stays stable is the partner company itself, the contract terms, and the rate card.

Per [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]], each sub-contracted [[Vehicle]] row points back to a Subcontractor.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `subcontractor_id` | string | PK |
| `name` | string | Display name |
| `contact_name` | string | Primary contact |
| `contact_phone` | string | |
| `commercial_license_ref` | string? | Partner's commercial transport licence reference (compliance) |
| `insurance_ref` | string? | Insurance policy reference |
| `insurance_expiry` | date? | For optional soft-block on contract expiry — see Open Items |
| `notes` | string | |

## Cardinality

- One Subcontractor → many [[Vehicle]] rows. One per `vehicle_type_id` the partner supplies. (E.g. Partner X supplies 22-wheel and 10-wheel → two Vehicle rows, both `ownership = sub_contracted`.)
- One Subcontractor → many [[Subcontractor Rate]] rows. One per `(subcontractor, vehicle_type, method)` time-versioned by close-and-replace.

The actual driver per trip is captured as denormalised fields on [[Work Order Driver]] (`external_driver_*`). Drivers are not modelled as standing rows under a Subcontractor in v1 — they rotate freely and the cost of dedup logic outweighs the analytics gain. See [[ADR-029]].

## Adding a Partner

```
1. INSERT Subcontractor row.
2. INSERT one Vehicle row per vehicle_type the partner supplies
   (ownership = sub_contracted, subcontractor_id = the new partner, plate_number = null).
3. INSERT SubcontractorRate row(s) for each vehicle_type with the agreed rate method.
```

VRP picks up the new slots automatically once the rates are in.

## Removing a Partner

```
1. UPDATE every related Vehicle row's status to out_of_service
   (or disposed if the relationship is permanently ended).
2. Close every active SubcontractorRate (effective_until = today).
```

Past Work Orders that referenced these Vehicles continue to read fine — the rows persist for trace; they just become unselectable for new assignments.

## Open Items

- **Insurance / licence soft-block.** Optional later: when assigning a sub-contract slot to a WO, soft-block if `insurance_expiry < wo.date` with a free-text override. Same pattern as [[ADR-026 — Vehicle Type License Requirement and Driver Filter]].
- **Per-trip volume reporting** ("how many trips with Partner X this month, by vehicle type"). Straight aggregation on [[Work Order]] joined through [[Vehicle]] to Subcontractor; no model change needed.
- **Driver standing record under a Subcontractor.** Promotable from the v1 denormalised approach if reporting on individual partner drivers becomes valuable.

## Related

- [[Vehicle]] — sub-contracted Vehicle rows reference Subcontractor
- [[Subcontractor Rate]] — per-trip rates by vehicle type
- [[Work Order]] — sub-contract WOs assign a sub-contracted Vehicle
- [[Work Order Driver]] — denormalised external driver fields capture the partner's driver per trip
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
