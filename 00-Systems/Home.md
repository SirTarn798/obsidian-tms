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
|`05-Module-Integrations`|How modules connect to each other|

## Start Here

- **[[Model — Order to Work Order Chain]]** — the central data model. The spine that solves the VRP traceability problem.

## Modules

- [[Cost Ledger — Overview]]
- [[Order Management — Overview]]
- [[Work Order — Overview]]
- [[Driver Management — Overview]]
- [[VRP — Overview]] _(placeholder)_
- [[Vehicle Maintenance — Overview]] _(placeholder)_

## Core Concepts

- [[Customer]]
- [[Service]] · [[Service Stop]] · [[Customer Location]]
- [[Quotation]] · [[Quotation Service]] · [[Price Rule]] · [[Allocation Rule]]
- [[Order]] · [[Order Line]] · [[Order Location]]
- [[Work Order]] · [[Work Order Stop]] · [[Work Order Vehicle]] · [[Work Order Driver]] · [[Route]]
- [[Vehicle]] · [[Vehicle Type]]
- [[Subcontractor]] · [[Subcontractor Rate]]
- [[Driver]] · [[Driver Salary]] · [[Driver Rate]]
- [[Cost Rule]] · [[Cost Event]] · [[Cost Per Km]] · [[Monthly Operational Cost]]

## Key Integration Points

- [[Integration — Quotation × Order]]
- [[Integration — Order × Work Order]]
- [[Integration — Order × Cost Ledger]]
- [[Integration — Vehicle Maintenance × Cost Ledger]]
- [[Integration — Driver × Cost Ledger]]
- [[Integration — User × Cost Ledger]]

## Tags Convention

- `#decision` — a design decision with reasoning
- `#module/cost-ledger` `#module/vrp` `#module/order-mgmt` etc.
- `#status/active` `#status/placeholder` `#status/deprecated`
- `#concept` — domain concept definition
- `#integration` — cross-module interaction