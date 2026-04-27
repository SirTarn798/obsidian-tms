# Conventions

This file collects two kinds of conventions: **Vault Conventions** (how notes are organised) and **Value Conventions** (cross-cutting domain rules that apply platform-wide).

---

## Value Conventions

Cross-cutting domain rules. Each is locked by an ADR; the rule lives here for quick reference.

### Multi-tenancy

Every **tenant-data** table carries a `group_id` column. System catalogue tables ([[Vehicle Type]], enum tables, platform-level config) do not. Isolation is enforced by service-layer filtering on `group_id`; row-level security is a deferred defense-in-depth measure. See [[ADR-031 — Multi-Tenancy via group_id on Tenant Data]].

### Currency

v1 is single-currency. All monetary fields are decimals in **THB (Thai Baht)**. No `currency` column on [[Price Rule]], [[Cost Rule]], [[Cost Event]], [[Driver Rate]], [[Order]], invoice, or report tables. Multi-currency is deferred. See [[ADR-031]] companion note (recorded in this conventions doc, not its own ADR — the decision is "don't add the dimension yet").

---

## Vault Conventions

### Note Naming

- Module overviews: `<Module Name> — Overview`
- Design decisions: `ADR-<NNN> — <Short Title>`
- Domain concepts: `<Concept Name>` (singular, e.g. "Cost Rule" not "Cost Rules")
- Integration notes: `Integration — <Module A> × <Module B>`
- Data models: `Model — <Entity Name>`

### Linking Rules

- Always link to domain concepts on first mention in a note: `[[Cost Rule]]`, `[[Order]]`, `[[Work Order]]`
- Always link to relevant ADRs when explaining _why_ something works a certain way
- Cross-module references should go through integration notes where possible

### Note Structure for Architecture Notes

```
# Title
> One-sentence summary of what this module/component does

## Context
Why does this exist? What problem does it solve?

## How It Works
The actual design — keep it focused on concepts and flow, not code

## Key Decisions
Links to relevant ADRs

## Connects To
Links to integration notes and other modules

## Open Questions
Things not yet decided
```

### For AI Assistants (Claude Code / MCP)

When reading this vault for context:

1. Start with the relevant `Overview` note in `01-Architecture`
2. Follow links to `03-Design-Decisions` for the _why_
3. Check `05-Module-Integrations` for how modules interact
4. Check `02-Domain-Concepts` for shared terminology

Design decisions are the most important notes — they explain _why_ things are the way they are, which prevents accidentally undoing intentional choices.