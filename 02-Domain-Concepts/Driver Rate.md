# Driver Rate

#concept #module/driver-mgmt #module/cost-ledger #status/active

> Compensation rate for a [[Driver]], parameterized by role, pay basis, and scope.

## Definition

A Driver Rate defines how much a Driver is paid for a given context. Rates are stored separately from the Driver record so they can change over time via close-and-replace ([[ADR-001 — Close and Replace, Never Mutate]]) and so the same Driver can earn different amounts depending on which role they fill, which route they run, or which customer they serve.

Monthly base salary is tracked as a sibling concept (see [[Integration — Driver × Cost Ledger]]); Driver Rate covers per-trip and per-km compensation only.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `rate_id` | string | Unique identifier |
| `driver_id` | string | Linked [[Driver]] |
| `role` | enum | `main` / `helper` |
| `pay_basis` | enum | `per_trip` / `per_km` |
| `scope` | enum | `default` / `service` / `customer` |
| `service_id` | string | Set when `scope=service`, else NULL. References [[Service]] (per-customer named route). |
| `customer_id` | string | Set when `scope=customer`, else NULL |
| `rate` | decimal | Amount in baht |
| `effective_from` | date | Inclusive |
| `effective_until` | date | Inclusive, NULL = open |

## Resolution Order

For a given `(driver, role, pay_basis)` and a [[Work Order Stop]] (resolved per stop, not per WO), the active rate is selected by **most-specific-wins**:

```
1. customer_id matches the stop's OrderLine.order.customer_id  → customer-scope rate
2. else service_id matches the stop's OrderLine.service_id      → service-scope rate
3. else                                                         → default-scope rate
4. else                                                         → no rate, contributes 0
```

Resolution is **override**, not stacking. A customer-scope rate of 7 fully replaces a default-scope rate of 5; the result is 7, not 12.

Per-stop resolution (rather than per-WO) is needed because VRP can place stops from multiple Services on one WO. For a `per_trip` WO-level pay basis where stops resolve to different rates, the resolver picks the rate matching each stop and sums; for `per_km` it weighs each stop by `wos.distance_actual_km`. See [[ADR-025 — DriverRate route_id Renamed to service_id]].

## Per-Trip vs Per-Km

- **per_trip** — flat amount per Work Order completed. Stop count is irrelevant.
- **per_km** — `rate × work_order.total_distance_actual_km` (planned distance as fallback if trip not yet complete).

The two pay bases are independent. A driver may have both `per_trip` and `per_km` rates active at once; both contribute to compensation on the same WO.

## Role Differentiation

Helper rates and main rates are stored as separate rows. A driver who works as `main` on WO-A and `helper` on WO-B gets:

- WO-A: `main` rates resolved
- WO-B: `helper` rates resolved

Monthly salary is **not** role-differentiated — see [[Integration — Driver × Cost Ledger]].

## Close and Replace

Rate changes follow [[ADR-001 — Close and Replace, Never Mutate]]:

```
Existing rate: per_trip / main / default = 500, effective_from=2025-01-01, effective_until=NULL

Raise to 600 effective 2026-05-01:
  1. UPDATE existing SET effective_until=2026-04-30
  2. INSERT new SET rate=600, effective_from=2026-05-01, effective_until=NULL
```

Past Work Orders continue to resolve to the historical rate by date.

## Examples

```
DriverRate(driver=DRV-001, role=main,   basis=per_trip, scope=default,                              rate=500)
DriverRate(driver=DRV-001, role=helper, basis=per_trip, scope=default,                              rate=200)
DriverRate(driver=DRV-001, role=main,   basis=per_km,   scope=default,                              rate=2.5)
DriverRate(driver=DRV-001, role=main,   basis=per_trip, scope=customer, customer_id=CUS-009,        rate=750)
DriverRate(driver=DRV-001, role=main,   basis=per_trip, scope=service,  service_id=SVC-A-bkk-cnx,   rate=650)
```

For a stop on `Service SVC-A-bkk-cnx` belonging to `Customer CUS-009`, with DRV-001 as main:
- Per-trip resolves to **750** (customer beats service beats default).
- Per-km resolves to **2.5** (only default exists).

## Related

- [[Driver]]
- [[Service]] — service-scope rates attach here ([[ADR-020]] makes Services per-customer)
- [[Work Order Driver]] — supplies role per WO
- [[Work Order]] — supplies WO context
- [[Work Order Stop]] — per-stop resolution unit
- [[Order Line]] — supplies service_id and (transitively) customer_id
- [[Integration — Driver × Cost Ledger]]
- [[ADR-001 — Close and Replace, Never Mutate]]
- [[ADR-020 — Service Is Per-Customer]]
- [[ADR-025 — DriverRate route_id Renamed to service_id]]
