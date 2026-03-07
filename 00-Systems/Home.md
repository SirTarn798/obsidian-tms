# Logistics Platform — Knowledge Base

This vault contains architecture decisions, domain concepts, and module designs for the logistics platform.

## How This Vault Is Organized

|Folder|Purpose|
|---|---|
|`00-System`|Vault conventions, tags, templates|
|`01-Architecture`|Per-module architecture notes (subfolder per module)|
|`02-Domain-Concepts`|Shared domain terms and definitions used across modules|
|`03-Design-Decisions`|ADRs (Architecture Decision Records) — the _why_ behind choices|
|`04-Data-Models`|Entity definitions, schemas, relationships|
|`05-Module-Integration`|How modules connect to each other|

## Modules

- [[Cost Ledger — Overview]]
- [[Order Management — Overview]] _(placeholder)_
- [[VRP — Overview]] _(placeholder)_
- [[Vehicle Maintenance — Overview]] _(placeholder)_
- [[Employee Management — Overview]] _(placeholder)_
- [[Work Order Income — Overview]] _(placeholder)_

## Key Integration Points

- [[Integration — Cost Ledger × Delivery Costing]]
- [[Integration — Vehicle Maintenance × Cost Ledger]]
- [[Integration — Employee Management × Cost Ledger]]

## Tags Convention

- `#decision` — a design decision with reasoning
- `#module/cost-ledger` `#module/vrp` `#module/order-mgmt` etc.
- `#status/active` `#status/placeholder` `#status/deprecated`
- `#concept` — domain concept definition
- `#integration` — cross-module interaction