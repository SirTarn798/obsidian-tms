# Driver

#concept #module/driver-mgmt #status/active

> A person who operates or assists on a [[Work Order]]. The unit of compensation in transport cost.

## Definition

A Driver is the operational human resource assigned to execute a [[Work Order]]. A Driver may serve as the **main driver** (operates the vehicle) or as a **helper** (assists with loading, stops, etc.) — see [[Work Order Driver]]. The same person may be main on one Work Order and helper on another.

Drivers are the only humans tracked as cost entities in this system. Other staff (controllers, planners, accountants) are not in scope for transport cost — see [[ADR-013 — Driver as Cost Entity, Not Employee]].

## Key Fields

| Field | Type | Description |
|---|---|---|
| `driver_id` | string | Unique identifier |
| `employee_code` | string | Unique short code, e.g. `DRV-001`. Used as `User.username` if mobile login enabled |
| `name` | string | Full name |
| `phone` | string | Contact number (not necessarily unique) |
| `license_no` | string | Driving license number — nullable (helpers may not hold one) |
| `license_classes` | array<string> | Held licence classes (e.g. `['B', 'C']`). Matched against [[Vehicle Type]]`.required_license_classes` for `main` assignment. Nullable. |
| `license_issued_date` | date | Nullable |
| `license_expiry_date` | date | Nullable. Expiry triggers soft-block on `main` assignment |
| `status` | enum | `active` / `inactive` |
| `created_at` | datetime | |
| `updated_at` | datetime | |

## License Tracking

Driver holds licence info directly. A daily check flags drivers whose `license_expiry_date` falls within 30 days for the controller dashboard.

**Soft-block rule** (per [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]): assigning a Driver as `main` on a [[Work Order]] is allowed silently when both:

- `license_expiry_date >= work_order.date`, **and**
- `license_classes ∩ vehicle_type.required_license_classes ≠ ∅`

Either condition failing triggers a warning + a required free-text override reason recorded on [[Work Order Driver]]. Helpers are not licence-checked.

The `main` driver dropdown defaults to filtering by these conditions; a "Show all drivers" toggle exposes the full list (and triggers the soft-block flow on selection).

## Compensation

A Driver may have:

- One monthly base salary — single rate, role-agnostic. See [[Driver Rate]] and [[Integration — Driver × Cost Ledger]].
- Per-trip rates — vary by `role` (main / helper) and `scope` (default / route / customer).
- Per-km rates — vary by `role` and `scope`.

A Driver who never works as main can still receive a monthly salary and helper-only rates.

## Authentication

A Driver row alone gives no system access. Mobile app login requires a paired [[User]] row with `User.driver_id` set and `role = driver-mobile`. See [[Authentication Scenarios]].

## Lifecycle

```
Driver hired
  → INSERT Driver (license, rates, monthly salary if any)
  → optionally INSERT User for mobile app

Driver assigned to Work Order
  → INSERT WorkOrderDriver(work_order_id, driver_id, role=main|helper)

Driver leaves
  → UPDATE Driver SET status=inactive
  → close DriverSalary / DriverRate via effective_until
  → soft-disable User (audit trail preserved)
```

## Related

- [[Work Order Driver]] — junction with role (main/helper)
- [[Driver Rate]] — per-trip / per-km compensation
- [[Driver Salary]] — monthly base, role-agnostic
- [[Work Order]] — assignment target
- [[Vehicle Type]] — supplies `required_license_classes` for the soft-block check
- [[Fuel Log]] — driver may submit fuel records via mobile app
- [[Authentication Scenarios]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — Driver × Work Order]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]
- [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]
