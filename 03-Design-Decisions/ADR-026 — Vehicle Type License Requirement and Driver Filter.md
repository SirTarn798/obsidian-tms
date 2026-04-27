# ADR-026 — Vehicle Type License Requirement and Driver Filter

#decision #module/driver-mgmt #module/work-order #status/active

## Status

Accepted. Refines license-check semantics from [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]] and earlier soft-block notes on [[Driver]] / [[Work Order Driver]].

## Context

The original license check was a single rule: if the assigned `main` driver's `license_expiry_date < work_order.date`, show a warning and let the controller proceed. This catches expired licenses but misses the more common everyday problem: the driver's licence **class** doesn't cover the assigned vehicle.

A driver licensed for a 4-wheel pickup cannot legally operate a 10-wheel rigid truck. Today the controller has to remember that mapping per driver. With dozens of drivers and several vehicle classes the mistake is easy and the consequences (fines, accident liability) are real.

There are two related operational asks:

1. **Filter UX.** When picking a `main` driver for a Work Order, the dropdown should default to drivers whose licence covers the assigned vehicle's type — not the full list.
2. **Override audit.** When the controller deliberately picks a driver outside the rule (expired licence, missing class), the system should record who did it, when, and why. The "why" is free text — the situations are too varied to enumerate.

The check should remain a **soft-block**, not a hard reject — operations sometimes legitimately need a one-off override and the controller is accountable for the call.

## Decision

**Vehicle Type carries the licence classes required to operate it. Driver carries the licence classes held. WorkOrderDriver carries audit fields for any override.**

### Schema

[[Vehicle Type]] (new concept doc):

| Field | Notes |
|---|---|
| `vehicle_type_id` | PK |
| `name` | Display name (4-wheel, 10-wheel, 18-wheel, ...) |
| `required_license_classes` | Array. Driver must hold **at least one** of these to drive this vehicle type without override. |

[[Driver]]:

| Field | Status | Notes |
|---|---|---|
| `license_classes` | **CHANGED** — was `license_type` string. Array of held classes (e.g. `['B', 'C']`). |
| `license_no`, `license_issued_date`, `license_expiry_date` | Keep |

[[Work Order Driver]] gains override audit fields, all NULL unless the override fires:

| Field | Notes |
|---|---|
| `license_override_by_user_id` | User who clicked "Proceed anyway". |
| `license_override_at` | Timestamp of override. |
| `license_override_reason` | Free text. Required when override fires. |

### Soft-Block Rule

On assigning a Driver as `main` to a Work Order with vehicle of type `T`:

```
allowed = (
    driver.license_expiry_date IS NOT NULL
    AND driver.license_expiry_date >= work_order.date
    AND (driver.license_classes ∩ T.required_license_classes) ≠ ∅
)
```

| Outcome | Behaviour |
|---|---|
| `allowed = true` | Save junction row silently. Override fields stay NULL. |
| `allowed = false` | Show warning describing the failed condition (expired vs. missing class). Require explicit "Proceed anyway" + non-empty `license_override_reason`. Record `*_by_user_id`, `*_at`, `*_reason`. |

Helpers are not licence-checked. They may not hold a licence at all.

### Filter UX

The `main` driver dropdown defaults to filtering by `(driver.license_classes ∩ T.required_license_classes) ≠ ∅` and `license_expiry_date >= work_order.date`. A "Show all drivers" toggle reveals the full list — picking from the unfiltered list triggers the soft-block flow above.

For helpers there is no filter — full driver list is shown.

### Reporting

Override rows are queryable for compliance review:

```sql
SELECT * FROM WorkOrderDriver
WHERE license_override_at IS NOT NULL
  AND license_override_at >= ?
ORDER BY license_override_at DESC
```

Override frequency by user/customer/period is the operational signal — frequent overrides for the same driver suggest their licence record needs updating; frequent overrides by the same controller suggest training is needed.

## Consequences

- **Good:** Class-mismatch is caught at assignment, not after a roadside check.
- **Good:** Filter UX cuts noise; controller picks from eligible pool by default.
- **Good:** Audit trail per override — who, when, why — supports compliance review.
- **Good:** Free-text reason captures novel situations the system can't anticipate.
- **Tradeoff:** `license_classes` as an array (vs. single class) requires the UI to render multi-class drivers correctly.
- **Tradeoff:** Vehicle Type catalogue must be maintained — adding a new class size means setting `required_license_classes`.
- **Tradeoff:** Free-text reasons can't be auto-categorised. Reporting reads them by hand or with later text mining.

## Related

- [[Driver]]
- [[Vehicle Type]]
- [[Work Order Driver]]
- [[Work Order]]
- [[Driver Management — Overview]]
- [[Integration — Driver × Work Order]]
- [[ADR-012 — Multi-Driver Work Order (Main + Helpers)]]
