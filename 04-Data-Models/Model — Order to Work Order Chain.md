# Model — Order to Work Order Chain

#model #module/order-mgmt #module/work-order #module/cost-ledger #status/active

> The central data model of the platform. The chain from Customer down to WorkOrderStop is the spine that solves the VRP traceability problem: customer-facing data never moves while VRP freely re-shapes the execution layer below.

## Why This Shape Exists

VRP merges and splits Work Orders for routing efficiency. A customer's "Route A" may end up as fragments across 5 different merged Work Orders. In the previous design this destroyed the trace — at billing time we could only see Orders, not what each Order actually executed against, and per-route profitability was an approximation at best.

This model fixes it by drawing a hard line:

- **Order layer** (Customer → Order → OrderLine → OrderLocation): the stable customer-facing record. Never moves. Owns revenue. Carries snapshots of pricing and allocation.
- **WorkOrder layer** (WorkOrder → WorkOrderStop): the volatile execution record. VRP moves freely here. Owns cost.
- **WorkOrderStop**: the movable join. The only thing that re-attaches when VRP re-plans.

Costs flow back from the WorkOrder layer to the OrderLine layer through this join, so per-route P&L is always derivable regardless of how VRP shuffled the trips.

## Diagram

```
                            ┌──────────────┐
                            │  Customer    │  *NEW concept doc*
                            │  ─default_invoice_policy
                            └──────┬───────┘
                                   │ owns
            ┌──────────────────────┼──────────────────────┐
            ▼                      ▼                      ▼
      ┌──────────┐          ┌──────────────┐      ┌─────────────────┐
      │ Service  │          │  Quotation   │      │ CustomerLocation│
      │ *per-cust│          │              │      │  (master)       │
      └────┬─────┘          └──────┬───────┘      └────────┬────────┘
           │ contains N            │ contains N             │
           ▼                       ▼                        │
      ┌──────────────┐    ┌────────────────────┐            │
      │ ServiceStop  │◄───┤ QuotationService   │            │
      │ ref Customer │    │ PK = (quotation,   │            │
      │  Location ───┼────►       service,     │            │
      └──────────────┘    │       vehicle_type)│            │
                          │ owns PriceRule     │            │
                          │ owns AllocationRule│            │
                          └─────────┬──────────┘            │
                                    │                       │
                  operator picks N ─┘                       │
                                    │                       │
                                    ▼                       │
                          ┌────────────────────┐            │
                          │      Order         │            │
                          │  +revenue          │            │
                          │  +billable_revenue │            │
                          │  +completed_at NEW │            │
                          │  +cost (derived)   │            │
                          │  status: pending → │            │
                          │  in_progress →     │            │
                          │  completed/cancelled│           │
                          └────────┬───────────┘            │
                                   │ contains N             │
                                   ▼                        │
                          ┌────────────────────┐            │
                          │     OrderLine      │            │
                          │  ├ PriceRule snap  │            │
                          │  ├ AllocRule snap  │            │
                          │  ├ revenue         │            │
                          │  ├ cost (derived)  │            │
                          │  ├ status (mirror) │            │
                          │  └ ref QS+Service  │            │
                          └────────┬───────────┘            │
                                   │ contains N             │
                                   ▼                        │
                          ┌────────────────────┐ snapshot   │
                          │   OrderLocation    │◄───────────┘
                          │  +cost (derived)   │
                          │  status flow       │
                          └────────┬───────────┘
                                   │  joined by
                                   ▼  (the movable join — VRP shuffles freely)
                          ┌────────────────────┐
                          │  WorkOrderStop     │
                          │  +cost (derived)   │
                          │  leg odometer      │
                          │  evidence          │
                          └────────┬───────────┘
                                   │  N belongs to 1
                                   ▼
                          ┌────────────────────┐
                          │    WorkOrder       │
                          │  cost = fixed_share│
                          │       + variable   │
                          │       + driver     │
                          │       + events     │
                          │  4 timestamps      │
                          │  vehicle, drivers  │
                          └────────────────────┘
```

## Layer Walkthrough

### Layer 0 — Master data (per Customer)

| Concept | Purpose |
|---|---|
| [[Customer]] | Owning party. Holds default invoice policy. *(New concept doc.)* |
| [[Customer Location]] | Physical sites the fleet operates to/from for this customer. Mutable master. |
| [[Service]] | The customer's named route — sequence of pickups and drops. **Per-customer**, not global. |
| [[Service Stop]] | Template stops within a Service. Reference CustomerLocation master. |

### Layer 1 — Catalog (Quotation)

| Concept | Purpose |
|---|---|
| [[Quotation]] | Commercial agreement with one Customer. |
| [[Quotation Service]] | Catalog entry. PK = `(quotation_id, service_id, vehicle_type_id)` — same Service can be priced differently per vehicle type. Owns one [[Price Rule]] and one [[Allocation Rule]]. |
| [[Price Rule]] | Pricing method + params. Methods: `flat`, `per_km` (uses `quoted_distance_km`, not VRP route), `per_weight`, `per_stop`, `tiered`. **`cost_plus` removed.** |
| [[Allocation Rule]] | How OrderLine revenue distributes across drop OrderLocations for ops-view analytics. |

### Layer 2 — Order (customer commitment, immutable trace)

| Concept | Purpose |
|---|---|
| [[Order]] | Customer's commitment for a job. Owns revenue. Status: `pending → in_progress → completed | cancelled`. |
| [[Order Line]] | One service instance — one row per selected QuotationService. Owns PriceRule + AllocationRule snapshots and `revenue`. Cost is derived from its OrderLocations. |
| [[Order Location]] | One stop. Snapshot of the master CustomerLocation at order time. Status flow includes `done | failed | cancelled | migrated`. |

### Layer 3 — Execution (volatile, VRP-managed)

| Concept | Purpose |
|---|---|
| [[Work Order]] | One trip — vehicle + drivers + date. Cost computed on read from Cost Ledger. Four-timestamp lifecycle. |
| [[Work Order Stop]] | The movable join. References an OrderLocation. Holds leg odometer (`distance_actual_km`), evidence, status. VRP creates and deletes these freely while OrderLocations stay put. |
| [[Work Order Driver]] | Junction with role (main / helper). |
| [[Route]] | Per-WO geometry only — polyline, planned distance. |

## Entity Field Deltas (target state)

These are the changes required from the current state of each concept doc. New ADRs will lock them in; concept docs will be edited to match.

### Customer **(new concept doc to create)**

| Field | Notes |
|---|---|
| `customer_id` | PK |
| `name`, `contact_name`, `contact_phone` | |
| `default_invoice_policy` | `all_or_nothing` \| `per_stop`. Inherited by Quotation unless overridden, then by Order. |

Currently referenced in [[ADR-011 — Partial Invoice Policy per Customer]] but not defined as a concept.

### Service **(rewrite — per-customer)**

| Field | Status | Notes |
|---|---|---|
| `customer_id` | **NEW** | Owning customer. |
| existing fields | keep | |

Drop the "global, shared across customers" framing throughout. ServiceStop references no longer ambiguous (each Service belongs to one Customer; its stops use that customer's CustomerLocations).

### QuotationService **(PK changes)**

| Field | Status | Notes |
|---|---|---|
| `vehicle_type_id` | **NEW** | Part of unique key |
| existing fields | keep | |

Unique constraint: `(quotation_id, service_id, vehicle_type_id)`. Same Service priced differently per vehicle class is first-class.

### PriceRule **(drop cost_plus, add quoted_distance_km)**

| Method | Status |
|---|---|
| `flat` | keep |
| `per_km` | **CHANGED** — params include `quoted_distance_km`; revenue computed from quoted distance, not VRP route distance. Decouples revenue from VRP entirely. |
| `per_weight`, `per_stop`, `tiered` | keep |
| `cost_plus` | **DROPPED** — broke "revenue knowable at order confirmation"; rare contract type handled via manual close + override. |

### AllocationRule **(scope mismatch fix)**

| Field | Status | Notes |
|---|---|---|
| `scope_type` | **DROPPED** | The enum was inconsistent with where rules actually live. |
| `quotation_service_id` | **NEW (nullable)** | Set when this is the catalog rule. |
| `order_line_id` | **NEW (nullable)** | Set when this is the snapshot on an OrderLine. |

Exactly one of the two scope columns is set per row.

### Order

| Field | Status | Notes |
|---|---|---|
| `completed_at` | **NEW** | Referenced by invoice query but never declared. Set when `status` transitions to `completed`. |
| `cost` | **NEW (derived, computed-on-read)** | `Σ OrderLine.cost` for per-Order P&L. |
| `status` | **CHANGED** | Add `cancelled` to enum. |

### OrderLine

| Field | Status | Notes |
|---|---|---|
| `cost` | **NEW (derived)** | `Σ OrderLocation.cost`. |
| `status` | **CHANGED** | Mirroring rule defined: `completed` if all OrderLocations terminal AND ≥1 done; `failed` if all terminal but none done; `in_progress` otherwise. |

### OrderLocation

| Field | Status | Notes |
|---|---|---|
| `cost` | **NEW (derived)** | One-to-one from its WorkOrderStop.cost. |

### WorkOrder

| Field | Status | Notes |
|---|---|---|
| Four timestamps | already added | `accepted_at`, `driving_started_at`, `delivery_ended_at`, `work_ended_at`. See [[Work Order]]. |
| `cost` (derived) | already added | Per Cost Ledger formula. |
| `status` | **CHANGED** | Add `cancelled` to enum (WO cancelled before driving started; timestamps remain null). |

### WorkOrderStop

| Field | Status | Notes |
|---|---|---|
| `cost` | **NEW (derived)** | Per the cost-back formula below. |

### DriverRate **(rename surfaced by the Cost Ledger pass)**

| Field | Status | Notes |
|---|---|---|
| `route_id` | **RENAME** | → `service_id`. Driver's "route-scope" pay attaches to the customer's Service, not the per-WO Route geometry. Resolution order unchanged (customer > service > default). |

## Cost-Back-to-OrderLine — Canonical Formula

```
WorkOrder.cost = fixed_cost_share + variable_cost + driver_cost + events_cost
                  ─── all four computed per Cost Ledger ───

Per WorkOrderStop wos in WO:
  wos.variable_cost = (fuel_cpk + maintenance_cpk + tire_cpk)
                      × wos.distance_actual_km

  N = count(WorkOrderStops in this WO)
  wos.fixed_share   = WO.fixed_cost_share / N
  wos.driver_share  = WO.driver_cost      / N    (incl. duration-based salary)
  wos.events_share  = WO.events_cost      / N

  wos.cost = wos.variable_cost + wos.fixed_share
           + wos.driver_share  + wos.events_share

OrderLocation.cost = its WorkOrderStop.cost  (one-to-one)
OrderLine.cost     = Σ OrderLocation.cost
Order.cost         = Σ OrderLine.cost
```

| Component | Allocation method | Why |
|---|---|---|
| `variable_cost` | Per-stop leg distance | Each stop's actual leg distance is known from odometer; matches physical fuel/maintenance consumption. |
| `fixed_cost_share` | By stop count | Fixed costs are paid for the trip, not the distance — distance is already in `cost_per_km`. Splitting by stops is simple and defensible. |
| `driver_cost` | By stop count | Same logic. Driver salary share is already proportional to WO duration; further per-stop refinement would need per-stop time tracking. |
| `events_cost` | By stop count | Default. Customer-specific extras tied to one stop can be refined later via optional `order_location_id` on CostEvent. |

Margin per OrderLine = `OrderLine.revenue − OrderLine.cost`. Both sides independently derived; no circular dependency on each other or on VRP.

## Trace Test (the original problem)

| Question | Query | VRP-stable? |
|---|---|---|
| Customer monthly invoice | `Σ Order.billable_revenue WHERE customer=X AND completed_at IN period` | ✓ |
| Per-route revenue (Customer A's "BKK→CNX") | `Σ OrderLine.revenue WHERE service_id=X AND order.customer=A` | ✓ |
| Per-route cost | `Σ OrderLine.cost WHERE service_id=X AND order.customer=A` | ✓ |
| Per-route margin | revenue − cost from above | ✓ |
| Cross-customer popularity of a route name | `count(OrderLine) GROUP BY service.name` | (cosmetic — Services are per-customer, so same name can repeat across customers; group by `service.name` not `service_id`) |

When VRP merges 6 OrderLines into 1 WorkOrder, each OrderLine still queries its own data through its OrderLocations through their WorkOrderStops. No information is lost in the merge or split.

## Snapshot vs Live Reference

The chain uses **snapshots at copy time**, not live references, at every layer where future mutations could corrupt past data:

| Layer | Snapshot of | Why |
|---|---|---|
| OrderLine | QuotationService's PriceRule and AllocationRule | Quotation price renegotiation must not change historical Order revenue. |
| OrderLocation | CustomerLocation's address and geo | Master address updates must not change historical orders. |
| OrderLine.quoted_distance_km | QuotationService PriceRule param | Revenue is locked at order confirmation, independent of VRP. |

Reverse direction (cost): **derived on read** from current data. WorkOrderStop cost is computed when queried, never stored as a fact. ADR-002 (compute on read) applies.

## Decisions Captured Here

The following ADRs are planned to lock in this model:

| ADR | Decision |
|---|---|
| ADR-020 | Service is per-customer |
| ADR-021 | QuotationService keyed by vehicle_type |
| ADR-022 | PriceRule per_km uses quoted_distance_km; cost_plus dropped |
| ADR-023 | Cost-back-to-OrderLine canonical formula |
| ADR-024 | Quotation amendment after `in_progress` (append-only OrderLine/OrderLocation post-WO-Stop creation) |
| ADR-025 | DriverRate route_id → service_id rename |

Existing ADRs that this model depends on:

- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-008 — WorkOrderStop as Movable Join]]
- [[ADR-009 — OrderLocation as Snapshot of CustomerLocation]]
- [[ADR-010 — Work Order Migration on Failure]]
- [[ADR-011 — Partial Invoice Policy per Customer]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-014 — Cost Classification]]
- [[ADR-015 — Per-WO Cost Events]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-019 — Driver Salary Allocation by Duration Share]]

## Refinements Held for v2

| Item | What it improves |
|---|---|
| Optional `order_location_id` on [[Cost Event]] | Per-stop attribution of customer-specific extras (forklift at one drop, document fee for one customer) instead of even split across WO stops. |
| Per-driver duration on [[Work Order Driver]] | Helper who joins for partial trip (loading only) bears salary share for actual hours, not full WO duration. |
| Refining cost-back split from "by stop count" to "by stop time" | If stop-count proves too coarse for fixed/driver/events allocation, weight by per-stop duration. |

## Related

- [[Order Management — Overview]]
- [[Work Order — Overview]]
- [[Cost Ledger — Overview]]
- [[Scenarios — Order to Invoice]]
- [[Integration — Quotation × Order]]
- [[Integration — Order × Work Order]]
- [[Integration — Order × Cost Ledger]]
