# Quotation

#concept #module/order-mgmt #status/active

> Contract offered to a customer. Defines which services are available and at what price — the order catalog for that customer.

## Definition

A Quotation is the commercial agreement with a customer. It contains a catalog of [[Quotation Service]]s — each one linking a [[Service]] (route template) to a [[Price Rule]] (what this customer pays). When operators create an [[Order]], they pick services from the customer's Quotation catalog.

Quotation does not own pricing directly. Pricing lives on [[Quotation Service]].

## Key Fields

| Field | Type | Description |
|---|---|---|
| `quotation_id` | string | Unique identifier |
| `customer_id` | string | Customer this was offered to |
| `status` | enum | `draft` → `sent` → `accepted` / `rejected` / `expired` |
| `valid_from` | date | |
| `valid_to` | date | |
| `default_invoice_policy` | enum | `all_or_nothing` or `per_stop` — propagates to Orders |
| `notes` | string | Contract terms, special conditions |

## Service Catalog

```
Quotation (Customer A, valid 2026)
  ├── QuotationService: "BKK → CNX" 4-wheel  → PriceRule flat 12,000
  ├── QuotationService: "BKK → CNX" 10-wheel → PriceRule flat 15,000
  ├── QuotationService: "BKK → LPG" 4-wheel  → PriceRule flat 12,000
  └── QuotationService: "ท่าเรือ → คลัง A" 18-wheel → PriceRule per_km 22 × quoted_distance_km
```

[[Quotation Service]] is keyed by `(quotation, service, vehicle_type)` per [[ADR-021 — QuotationService Keyed by Vehicle Type]] — same Service can appear once per supported vehicle type at different prices.

Each Quotation references **this customer's own [[Service]]s only** — Services are per-customer per [[ADR-020 — Service Is Per-Customer]].

## Order Creation Flow

1. Operator opens Order form, selects Customer
2. System loads this Quotation's Quotation Services as a menu
3. Operator picks a `(service, vehicle_type)` combination
4. Each selection → one [[Order Line]] created
5. PriceRule + AllocationRule + `quoted_distance_km` copied from QuotationService to OrderLine
6. Service stops copied as [[Order Location]]s (adjustable until WO Stop created)

Orders reference `quotation_id` for traceability but do not depend on Quotation after creation. Quotation price changes (close-and-replace) do not affect existing Orders. Mid-flight amendments append-only per [[ADR-024 — Quotation Amendment After In-Progress]].

## Related

- [[Customer]] — owner
- [[Quotation Service]] — service+vehicle-type+price entries in this catalog
- [[Service]] — per-customer route templates referenced
- [[Order]] — orders placed under this quotation
- [[Price Rule]] — lives on Quotation Service, not here
- [[Integration — Quotation × Order]]
- [[ADR-020 — Service Is Per-Customer]]
- [[ADR-021 — QuotationService Keyed by Vehicle Type]]
- [[ADR-024 — Quotation Amendment After In-Progress]]
