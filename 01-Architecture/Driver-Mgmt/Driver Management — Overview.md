# Driver Management — Overview

#module/driver-mgmt #status/active

> Owns Driver records, compensation rates, license tracking, mobile authentication, and fuel logs. The transport-cost view of human resources.

## Context

This module replaces the previous "Employee Management" module. The earlier design forced every system user to be an Employee, requiring HR-style fields irrelevant to transport. The new design separates concerns:

- **Driver** — the operational human resource that executes Work Orders. The only human-side cost entity in transport.
- **User** — the authentication record. Anyone who logs into the web or mobile app. May be a controller, planner, accountant, CEO, or driver.

Controllers, planners, and other office staff are tracked as Users only. Their salaries **are** captured in the [[Cost Ledger]] (as `entity_type=user` rules) but flow into [[Monthly Operational Cost]] as overhead — they are not allocated to individual Work Orders. See [[ADR-013 — Driver as Cost Entity, Not Employee]] and [[Integration — User × Cost Ledger]].

## How It Works

### Layer 1 — Identity

| Concept | Purpose |
|---|---|
| [[Driver]] | Operational resource. Holds license, contact, compensation rates. |
| User (auth) | Login credentials. `User.driver_id` links a User to a Driver when mobile login is enabled. |

A controller has a User row only. A driver-only person has a Driver row only. A driver who uses the mobile app has both, linked via `User.driver_id`. See [[Authentication Scenarios]].

### Layer 2 — Compensation

| Concept | Purpose |
|---|---|
| Driver monthly salary | Single base rate per Driver, role-agnostic. Stored as a [[Cost Rule]] with `entity_type=driver, frequency=monthly`. |
| [[Driver Rate]] | Per-trip and per-km rates. Vary by `role` (main/helper) and `scope` (default/route/customer). |

A Driver's pay on a Work Order is the sum of:
1. Daily share of monthly salary (proration per [[ADR-003 — Month-by-Month Proration]])
2. Per-trip rate resolved by role + scope
3. Per-km rate resolved by role + scope, multiplied by actual distance

See [[Integration — Driver × Cost Ledger]] for the calculation in detail.

### Layer 3 — Assignment

A Work Order has one **main** driver and any number of **helpers**. Both are stored in [[Work Order Driver]]. See [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]].

Three soft-block checks fire at assignment time, all following the same pattern (warning + required free-text reason + audit fields on the affected row):

| Check | Scope | Reference |
|---|---|---|
| Licence (expiry + class match against [[Vehicle Type]]) | `main` only | [[ADR-026]] |
| Driver-overlap with another non-cancelled WO | any role | [[ADR-028]] |
| Vehicle-overlap with another non-cancelled WO | vehicle (recorded on WO) | [[ADR-028]] |

The `main` driver dropdown defaults to filtering by licence eligibility (expiry + class match for the assigned vehicle type). A "Show all drivers" toggle exposes the full list and triggers the soft-block flow on selection. Helpers are not licence-checked.

**Mid-trip junction mutation** ([[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]):

| Change | While `in_progress` |
|---|---|
| Add / remove / swap helper | Allowed. Records `added_at` / `removed_at`. Per-helper duration share derives from these. |
| Change main, change vehicle | Disallowed → use migration ([[ADR-010]]). |

### Layer 4 — Fuel Logging

Drivers submit [[Fuel Log]] entries from the mobile app. Each entry inserts a [[Cost Event]] keyed to the **vehicle**, not the Work Order. Fuel cost flows through the vehicle pool, not direct WO charging. See [[Integration — Fuel Log × Cost Ledger]].

### Layer 5 — License Tracking

Driver records hold licence fields: number, **classes held** (array — e.g. `['B', 'C']`), issued/expiry dates. A daily check flags drivers whose `license_expiry_date` falls within 30 days for the controller dashboard.

Assigning a Driver as **main** on a Work Order with vehicle of type `T` is silent when `driver.license_expiry_date >= wo.date AND (driver.license_classes ∩ T.required_license_classes) ≠ ∅`. Either condition failing triggers the soft-block flow — warning + required free-text override reason recorded on [[Work Order Driver]]. See [[ADR-026 — Vehicle Type License Requirement and Driver Filter]].

Helpers are not licence-checked.

## Key Design Principles

1. **Driver is operational, not HR.** Only fields needed for transport (licence, contact, rates).
2. **Auth is separate.** A User row carries credentials; a Driver row carries operational data. Linked by FK only when needed.
3. **Controllers are not per-WO cost.** Office staff salary is tracked in the Cost Ledger as overhead and rolls up into [[Monthly Operational Cost]], but is never charged to a specific Work Order.
4. **Helpers are paid.** A WO can have multiple paid people; the cost model accounts for all, with per-helper duration share for mid-trip joins/leaves.
5. **Most-specific-wins rate resolution.** Customer-scope beats service-scope beats default. Override, not stacking. Resolved per stop ([[ADR-025]]).
6. **Soft-block, with audit.** Licence, driver-overlap, vehicle-overlap all surface as warnings + required free-text reason. Frequency reports drive process improvement.
7. **Main change is heavy, helper change is light.** Main and vehicle changes go through migration. Helper add/remove/swap is in-place, even mid-trip.
8. **Driver assignment is the user's responsibility.** VRP picks vehicle (and may use vehicle availability + capacity); driver selection stays manual.

## Key Decisions

- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-001 — Close and Replace, Never Mutate]] — applies to rate and salary changes
- [[ADR-003 — Month-by-Month Proration]] — applies to monthly salary
- [[ADR-019 — Driver Salary Allocation by Duration Share]]
- [[ADR-025 — DriverRate route_id Renamed to service_id]]
- [[ADR-026 — Vehicle Type License Requirement and Driver Filter]]
- [[ADR-027 — Helper-Only Mid-Trip Junction Mutation]]
- [[ADR-028 — Driver and Vehicle Overlap Soft-Block]]

## Connects To

- [[Integration — Driver × Cost Ledger]] — driver compensation, charged per WO
- [[Integration — User × Cost Ledger]] — office staff salary, overhead bucket
- [[Integration — Driver × Work Order]] — assignment and license check
- [[Integration — Fuel Log × Cost Ledger]] — fuel cost events
- [[Authentication Scenarios]] — user/driver login flows
- [[Monthly Operational Cost]]
- [[Work Order — Overview]]
- [[Cost Ledger — Overview]]

## Open Questions

- Sub-contractor drivers — separate flag and rate model, deferred.
- Self-service password reset for drivers (SMS OTP) — deferred to v2.
- Licence hard-block policy switch — soft-block today; business may later opt to convert to hard-block per vehicle type.
- Driver availability / leave / weekly schedule — currently manual (controller knows). Could be modelled if VRP later gains driver awareness.
