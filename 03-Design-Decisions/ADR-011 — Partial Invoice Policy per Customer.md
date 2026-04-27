# ADR-011 — Partial Invoice Policy per Customer

#decision #module/order-mgmt #status/active

## Status

Accepted

## Context

When some stops in an [[Order]] fail or are cancelled, the business must decide: do we bill the customer for what was successfully delivered, or do we only bill when everything is complete? Different customers or contract types require different answers. Additionally, operators sometimes need to force-close an order and bill regardless of stop status.

The system must also support multiple companies with different default behaviors.

## Decision

**Invoice policy is configurable per Customer, overridable per Quotation. Manual close unlocks user confirmation of billable amount.**

### Policy Levels (precedence: Quotation → Customer)

```
Customer.default_invoice_policy: 'all_or_nothing' | 'per_stop'
Quotation.invoice_policy:        same | NULL (inherit from Customer)
Order.effective_invoice_policy = Quotation.invoice_policy ?? Customer.default_invoice_policy
```

### Policy Behavior

**`all_or_nothing`**
- `billable_revenue = order.revenue` if all eligible (drop) stops are `done`
- `billable_revenue = 0` otherwise
- Use when contract requires full delivery confirmation before billing

**`per_stop`**
- `billable_revenue = Σ order_location.allocated_share WHERE status=done AND stop_type=drop`
- Use when partial delivery has partial value (e.g., milk-run with independent stops)

### Manual Close

When an operator closes an order manually (some stops failed/cancelled):

1. System computes `billable_revenue` per effective policy and shows to operator
2. Operator may accept computed value or enter an override (e.g., "customer agreed to pay full despite missed stop")
3. `order.closure_type = manual`
4. `order.closure_note` = reason
5. `order.billable_revenue_source = computed | manual_override`

Manual close allows billing even when policy would otherwise produce 0. The audit trail (`closure_type`, `closure_note`, `billable_revenue_source`) records the decision.

### Eligibility

Only `drop` stops count toward billable revenue computation, consistent with [[Allocation Rule]] eligible_stop_types.

## Consequences

- **Good:** Flexible enough for different customer contracts and company policies
- **Good:** Manual close with audit trail gives operators escape hatch without hiding decisions
- **Good:** Policy inheritance keeps setup simple (set once on Customer, override only when needed)
- **Tradeoff:** `billable_revenue_source = manual_override` must be surfaced in invoice UI so finance can review overrides
