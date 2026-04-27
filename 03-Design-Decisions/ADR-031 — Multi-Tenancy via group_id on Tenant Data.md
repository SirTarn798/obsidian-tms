# ADR-031 — Multi-Tenancy via group_id on Tenant Data

#decision #status/active

## Status

Accepted. Establishes the platform-wide rule that the system is multi-tenant from v1, with tenant isolation enforced by a `group_id` column on every tenant-data table.

## Context

The platform may host several operating companies on one deployment. Each tenant's data must be invisible to every other tenant by default — customers, orders, vehicles, drivers, invoices, cost events, all of it. A single-tenant assumption baked into the schema is expensive to retrofit: every table touched by a query, every joined view, every analytics report has to be revised.

Two threads make multi-tenancy a v1 concern rather than a deferred one:

1. **Sales reality.** The platform is expected to be sold to multiple operating companies. Tenant isolation can't be the v2 problem.
2. **Some entities are platform-shared.** [[Vehicle Type]] is a small reference catalogue (4-wheel, 10-wheel, 22-wheel, …) shared across tenants. Adding a `group_id` to those would force every tenant to maintain their own copy of an effectively universal list.

Two cross-cutting choices anchor the design:

- **Where `group_id` lives.** On every tenant-data row, including denormalised onto child tables for query efficiency. Joining up the chain at every read is needlessly expensive and brittle.
- **How isolation is enforced.** Service-layer filtering is the v1 mechanism. Postgres row-level security (RLS) is a defense-in-depth deferred to a later phase.

## Decision

**Every tenant-data table carries a `group_id` column. System catalogue tables do not. Isolation is enforced by mandatory `group_id` filtering at the service / repository layer.**

### Tenant-Data vs System Tables

| Category | Carries `group_id`? | Examples |
|---|---|---|
| Tenant data | Yes | [[Customer]], [[Customer Location]], [[Quotation]], [[Quotation Service]], [[Order]], [[Order Line]], [[Order Location]], [[Service]], [[Service Stop]], [[Work Order]], [[Work Order Stop]], [[Work Order Vehicle]], [[Work Order Driver]], [[Vehicle]], [[Driver]], [[Driver Rate]], [[Driver Salary]], [[Subcontractor]], [[Subcontractor Rate]], [[Cost Rule]], [[Cost Event]], [[Fuel Log]], [[Stop Evidence]], User, Invoice |
| System (platform-shared) catalogue | No | [[Vehicle Type]], licence-class enum tables, cost-classification enum tables, platform-level configuration |

Rule of thumb: if a row belongs to a specific operating company, it has `group_id`. If it is reference data the platform owns, it does not.

### Denormalisation

`group_id` is denormalised onto every tenant-data table — including children whose `group_id` could in principle be derived by joining up the parent chain. Reasons:

- Every list / report / analytics query starts with a `WHERE group_id = ?` filter. Reading from the child row directly avoids N upward joins.
- Indexes on `(group_id, …)` are first-class on every table.
- Cross-tenant data leaks through inadvertent joins are caught at the leaf, not deep in a chain.

Denormalised `group_id` values must be consistent across the chain. Foreign-key constraints don't enforce this — service-layer logic and tests do. A migration tool can periodically validate consistency.

### Enforcement

**Service-layer filter (v1).** Every repository / data-access function takes the current tenant's `group_id` from the request context (set at authentication) and includes it in every WHERE clause. There is no "no-tenant" data-access path; queries that omit `group_id` are a code-review and lint-time failure.

**Row-level security (deferred).** Postgres RLS, with the tenant's `group_id` set on the connection via `SET LOCAL`, is a strict defense-in-depth measure. Deferred to a later phase to reduce v1 cognitive load and migration overhead. The design does not preclude it — denormalised `group_id` is exactly what RLS policies need.

### Sub-Contracted Vehicles

Sub-contracted [[Vehicle]] supply slots and [[Subcontractor]] rows are tenant-scoped — each tenant has its own partner relationships. Both carry `group_id`. The Vehicle Type those slots reference is platform-shared (no `group_id` on Vehicle Type).

### Cross-Tenant Sharing

The current design has **no** cross-tenant sharing of tenant data. If a future case arises (e.g. a parent company viewing aggregated data across subsidiaries), it goes through a dedicated reporting layer with explicit permission, not by relaxing the isolation rule.

## Consequences

- **Good:** Tenant isolation is consistent and queryable from day one.
- **Good:** Multi-tenant onboarding is data-only — no schema rework per tenant.
- **Good:** Denormalised `group_id` keeps queries simple and indexes flat.
- **Good:** Future RLS adoption is straightforward — the column is already there.
- **Tradeoff:** Every tenant-data table grows by one column. Indexed on `(group_id, …)` everywhere; some storage overhead. Acceptable.
- **Tradeoff:** Service-layer filtering can be bypassed by direct SQL. Mitigated by code review, linters that flag missing `group_id` filters, and the RLS deferral path.
- **Tradeoff:** Denormalised `group_id` consistency is a service-layer invariant, not an FK. A bug that writes a child with the wrong `group_id` can leak. Mitigation: writes always copy from parent; periodic consistency check.

## Open Items

- **RLS adoption.** When (and whether) to roll out Postgres RLS as defense-in-depth. Currently deferred; revisit when the platform has more than three tenants or when a regulator requires hardened isolation.
- **Consistency audit job.** A scheduled job that walks parent → child chains and asserts `group_id` matches. Not a v1 blocker; the service layer is the first defense.

## Related

- [[Value Conventions]] — short-form rule
- [[Customer]] · [[Order]] · [[Work Order]] · [[Vehicle]] · [[Driver]] — all tenant-scoped, carry `group_id`
- [[Vehicle Type]] · system catalogues — platform-shared, no `group_id`
