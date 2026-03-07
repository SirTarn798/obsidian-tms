# Vault Conventions

## Note Naming

- Module overviews: `<Module Name> — Overview`
- Design decisions: `ADR-<NNN> — <Short Title>`
- Domain concepts: `<Concept Name>` (singular, e.g. "Cost Rule" not "Cost Rules")
- Integration notes: `Integration — <Module A> × <Module B>`
- Data models: `Model — <Entity Name>`

## Linking Rules

- Always link to domain concepts on first mention in a note: `[[Cost Rule]]`, `[[Delivery]]`
- Always link to relevant ADRs when explaining _why_ something works a certain way
- Cross-module references should go through integration notes where possible

## Note Structure for Architecture Notes

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

## For AI Assistants (Claude Code / MCP)

When reading this vault for context:

1. Start with the relevant `Overview` note in `01-Architecture`
2. Follow links to `03-Design-Decisions` for the _why_
3. Check `05-Module-Integration` for how modules interact
4. Check `02-Domain-Concepts` for shared terminology

Design decisions are the most important notes — they explain _why_ things are the way they are, which prevents accidentally undoing intentional choices.