---
tags: [concept, module/order-mgmt, status/active]
---

# Allocation Rule

#concept #module/order-mgmt #status/active

> How an [[Order Line]]'s revenue distributes across its drop [[Order Location]]s. Ops-view display only — never used for invoicing directly, but feeds `per_stop` invoice policy.

## Definition

Allocation Rule determines `order_location.allocated_share` — the portion of OrderLine revenue attributed to each drop stop. Used to compute [[Work Order]] `display_revenue`, route profitability, and the `per_stop` invoice policy on partial completion.

Revenue always belongs to the [[Order Line]]. Allocation is the lens that makes per-stop analytics and partial-stop billing possible.

## Key Fields

| Field | Type | Description |
|---|---|---|
| `allocation_rule_id` | string | |
| `quotation_service_id` | string | Set when this is the catalog rule (nullable). |
| `order_line_id` | string | Set when this is the snapshot on an OrderLine (nullable). |
| `method` | enum | See methods below |
| `eligible_stop_types` | array | Default: `['drop']` — pickups always get 0 |

**Constraint:** exactly one of `quotation_service_id` or `order_line_id` is set per row. Catalog rule lives on the [[Quotation Service]]; the snapshot copy lives on the [[Order Line]]. (Replaces the previous `scope_type` enum, which was inconsistent with where rules actually attached.)

## Methods

| Method | Formula | Requirement |
|---|---|---|
| `equal` | `revenue / count(eligible stops)` | None. Default. |
| `by_distance` | `(leg_distance / total_eligible_distance) × revenue` | Solved [[Route]] required |
| `by_weight` | `(stop.weight_kg / Σ weight_kg) × revenue` | `weight_kg` on OrderLocation |
| `by_quoted_sub_price` | Direct per-stop price from quotation line items | Manual setup |

**Default:** `equal`. Upgrade to `by_distance` when Route is solved.

## Pickup Exclusion

Pickup stops always receive `allocated_share = 0` regardless of method. Revenue represents value delivered at drop points, not the act of picking up. Pickups still incur **cost** through the cost-back formula ([[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]) — only the revenue side excludes them.

## Compute Timing

`allocated_share` is computed after the [[Price Rule]] gives `OrderLine.revenue`, then stored on each drop [[Order Location]]. Re-computed if OrderLine revenue changes or if drop OrderLocations are added/cancelled (per [[ADR-024 — Quotation Amendment After In-Progress]]).

## Scope Within an OrderLine

Allocation distributes revenue **within a single OrderLine** to its drop stops. Cross-OrderLine allocation does not exist — each OrderLine is a separate priced commitment. The Order-level invoice computation aggregates `Σ OrderLine.billable_revenue` across lines, where each line independently applies the policy and allocation.

## Related

- [[Quotation Service]] — catalog scope
- [[Order Line]] — snapshot scope; provides revenue to distribute
- [[Order Location]] — stores `allocated_share`
- [[Price Rule]] — provides the revenue
- [[Work Order]] — consumes allocations for `display_revenue`
- [[ADR-006 — Revenue Lives on Order, Not Work Order]]
- [[ADR-011 — Partial Invoice Policy per Customer]]
- [[ADR-023 — Cost-Back-to-OrderLine Canonical Formula]]
- [[ADR-024 — Quotation Amendment After In-Progress]]
