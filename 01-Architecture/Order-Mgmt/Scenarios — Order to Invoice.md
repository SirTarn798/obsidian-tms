# Scenarios — Order to Invoice

#module/order-mgmt #module/work-order #status/active

> End-to-end walkthroughs of the Order → Work Order → Invoice flow across common and edge cases.

---

## Entity quick reference

```
Customer → Quotation → QuotationService (service + PriceRule)
                              ↓ (operator selects at order time)
                          Order → OrderLine → OrderLocation → CustomerLocation (master)
                                                    ↕ (via WorkOrderStop join)
                                              WorkOrder → WorkOrderStop → Route
                                              WorkOrder → Vehicle, Driver

Service (global template) → ServiceStop
QuotationService → PriceRule, AllocationRule  (per customer price)
OrderLine        → PriceRule snapshot, AllocationRule snapshot (copied at creation)
```

---

## Scenario 1 — Happy path, single service order

**Setup:** Customer A has Quotation with QuotationService "BKK→CNX", PriceRule flat 15,000, AllocationRule equal.

```
1. Operator creates Order for Customer A
   - Picks QuotationService "BKK→CNX"
   - OrderLine OL-A created:
       PriceRule snapshot: flat 15,000
       AllocationRule: equal, eligible=['drop']
   - OrderLocations copied from ServiceStops:
       OL1 (pickup BKK): allocated_share = 0
       OL2 (drop CNX):   allocated_share = 15,000
   - OrderLine.revenue = 15,000
   - Order.revenue = 15,000, status=pending

2. VRP assigns to Vehicle V1, Driver D1
   - WorkOrder WO-1 created
   - Route R1 created (696 km)
   - WO Stops: WO-1→OL1 seq=1, WO-1→OL2 seq=2
   - Order.status = in_progress

3. Driver executes
   - WO Stop OL1: done, actual_arrival recorded
   - WO Stop OL2: done, actual_arrival recorded
   - WO-1.status = completed
   - Order.status = completed (auto)

4. Invoice (end of month)
   - policy=all_or_nothing, all drops done
   - billable_revenue = 15,000
   - Invoice: Customer A → 15,000 THB

5. Ops view (display only)
   - WO-1.display_revenue = 15,000 (OL2 done, share=15,000)
   - WO-1.cost = fixed share + (cost_per_km × 696) + driver cost
   - WO-1.display_margin = display_revenue − cost
```

---

## Scenario 2 — Multi-service order

**Setup:** Customer A orders two services in one Order.

```
Quotation services available:
  QS-1: "BKK→CNX" → flat 15,000
  QS-2: "BKK→LPG" → flat 12,000

Operator creates Order, picks both:

OrderLine A (from QS-1):
  OL1 pickup BKK: share=0
  OL2 drop CNX:   share=15,000
  revenue = 15,000

OrderLine B (from QS-2):
  OL3 pickup BKK: share=0
  OL4 drop LPG:   share=12,000
  revenue = 12,000

Order.revenue = 27,000

VRP creates WO-1 covering all 4 stops (efficient route BKK→CNX→LPG or similar)

All done:
  WO-1.display_revenue = 15,000 + 12,000 = 27,000
  Invoice: Customer A → Order → 27,000 THB (one line item)
```

---

## Scenario 3 — VRP merges orders from two customers

**Setup:** Two orders, different customers, VRP combines onto one truck.

```
Order O1 (Customer A): OL1 pickup BKK, OL2 drop CNX → revenue 15,000
Order O2 (Customer B): OL3 pickup BKK, OL4 drop LPG → revenue 12,000

WO-1: V1, D1
WO Stops: OL1 seq=1, OL3 seq=2, OL2 seq=3, OL4 seq=4

Revenue (canonical):
  Customer A invoice: O1 → 15,000
  Customer B invoice: O2 → 12,000
  (Work Order never appears on invoice)

Ops view:
  WO-1.display_revenue = OL2.share + OL4.share = 27,000
  WO-1.cost = cost of combined route
```

---

## Scenario 4 — VRP re-plan before departure

```
WO-1 assigned: [O1 stops, O2 stops]. Not departed.
New O3 arrives. VRP re-plans for better solution.

Actions:
  Delete WO Stops for O2 from WO-1
  Create WO-2 with O2 + O3 stops, new Route
  WO-1 updated to O1 stops only, new Route

Orders O1, O2, O3: untouched. Revenue: untouched.
```

---

## Scenario 5 — Partial failure, Migrate

```
WO-1 covers: OL1(O1 pickup), OL2(O1 drop), OL3(O2 pickup), OL4(O2 drop)
OL1=done, OL2=done. Accident. OL3, OL4 unfinished.

Migrate:
  WO-1: status=closed_partial
  WO Stop OL3: migrated, WO Stop OL4: migrated
  OrderLocation OL3, OL4: migrated → planned

  WO-2 created: V2, D2, migrated_from=WO-1
  New WO Stops for OL3, OL4 on WO-2

O1: completed. billable_revenue=15,000 ✓
O2: in_progress until WO-2 finishes

Cost:
  WO-1.cost = cost of partial trip
  WO-2.cost = cost of completing remaining stops

Ops view:
  WO-1.display_revenue = OL2.share = 15,000 (only O1 drop done by WO-1)
  WO-2.display_revenue = OL4.share = 12,000 (after WO-2 completes)
```

---

## Scenario 6 — Per-stop invoice policy, partial delivery

**Setup:** Customer C has `invoice_policy=per_stop`. Order: 1 pickup + 4 drops, flat 20,000, AllocationRule equal.

```
OrderLocations:
  OL1 pickup:  share=0
  OL2 drop:    share=5,000
  OL3 drop:    share=5,000
  OL4 drop:    share=5,000
  OL5 drop:    share=5,000

Execution: OL1–OL4 done. OL5 cancelled.

billable_revenue (per_stop):
  = Σ allocated_share WHERE status=done AND stop_type=drop
  = 5,000 + 5,000 + 5,000 = 15,000

Invoice: Customer C → 15,000 THB
OL5: may be re-ordered as new Order later.
```

---

## Scenario 7 — Manual close with user override

**Setup:** Customer has `all_or_nothing`. One stop failed. Customer agreed to pay full.

```
Order.revenue = 20,000, 3 drops
  OL1 done → 6,667
  OL2 done → 6,667
  OL3 failed → 6,667

Computed billable (all_or_nothing) = 0 (OL3 failed)

Operator clicks "Close & Bill":
  System shows computed=0
  Operator enters override: 20,000
  closure_note: "Customer accepted partial, agreed to full payment"

Order.closure_type = manual
Order.billable_revenue = 20,000
Order.billable_revenue_source = manual_override
Order.status = completed

Invoice: 20,000 (manual_override flagged for finance review)
```

---

## Scenario 8 — Operator adjusts price at order time

**Setup:** QuotationService default flat 15,000. Customer negotiates one-off 13,500 for this order.

```
Operator creates Order, picks QS "BKK→CNX":
  OrderLine created with PriceRule snapshot: flat 15,000

Operator edits PriceRule params: flat → 13,500
  OrderLine.revenue = 13,500
  OrderLine.price_manually_adjusted = true
  Order.revenue = 13,500

QuotationService PriceRule: unchanged (still 15,000 for future orders)

Invoice: 13,500 (manual adjustment visible for review)
```

---

## Summary — what each layer owns

| Question | Answer lives in |
|---|---|
| What services can customer order? | Quotation → QuotationService catalog |
| What did customer order this time? | Order + OrderLines |
| Which service template? | OrderLine.service_id → Service |
| What did we agree to charge? | OrderLine.revenue (PriceRule snapshot) |
| Total order revenue | Order.revenue = Σ OrderLine.revenue |
| What do we actually bill? | Order.billable_revenue |
| Who drove, what route? | WorkOrder + Route |
| Which stops did which WO cover? | WorkOrderStop |
| How much did trip cost us? | WorkOrder.cost (Cost Ledger) |
| Route P&L estimate? | WorkOrder.display_margin (ops only) |
| Customer invoice? | Σ Order.billable_revenue by customer + period |
