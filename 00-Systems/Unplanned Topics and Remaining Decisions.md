# Unplanned Topics and Remaining Decisions

#status/active

> Topics raised during design review but deferred. Each entry frames the **problem** and the **decision needed** so the discussion can pick up cold without re-deriving context.

Groups in rough priority. The current focus has been Cost Ledger → Quotation/Service/Order → Driver/User. Everything below is the next wave.

---

## Driver / User

### Driver availability / schedule

**Problem.** Today driver assignment is fully manual; controllers carry availability in their head. As soon as VRP gains driver-aware assignment (or once roster scales beyond what a controller can mentally track), an availability model is needed.

**Decision needed.** Scope: just leave periods (`(driver_id, from, to, reason)` rows) is enough for "don't assign on holiday" UX. A full weekly schedule (`day_of_week, start_time, end_time`) is needed only if VRP picks drivers automatically. Currently driver picking is the user's responsibility; this stays deferred until VRP changes.

### Self-service password reset

**Problem.** Driver password reset today goes through a controller. Drivers in the field who lock themselves out of the mobile app are blocked until office hours.

**Decision needed.** SMS OTP flow vs. email OTP vs. controller-only. SMS OTP is the natural fit (drivers always have a phone) but pulls in an SMS provider integration. Deferred to v2 in the Driver Mgmt overview.

### Licence hard-block policy switch

**Problem.** [[ADR-026]] sets licence/class match as soft-block. A regulator audit or insurance contract may later require hard-block — no override possible.

**Decision needed.** When (if) to flip the switch. Options:

- **Per-tenant config flag.** `licence_enforcement = soft | hard`.
- **Per-vehicle-type policy.** Some classes (e.g. dangerous goods) hard-block; others soft.
- **Defer entirely.** Wait until a real driver makes the ask.

No urgency; record only when business signals it.

---

## Smaller spec gaps

### Per-trip rate semantics on cancelled / partial WO

**Problem.** [[Driver Rate]] `per_trip` is a flat amount per Work Order. What if the WO is cancelled before driving started, or closes as `closed_partial` after a few stops? Does the driver earn the full trip rate, half, nothing? No documented rule.

**Decision needed.** Rate-per-status table:

- `cancelled` (no driving) → 0
- `closed_partial` after migration → full per-trip on the closed WO + full per-trip on the migrated WO (double-counts if not careful) **OR** prorated per-stop **OR** flat half on the partial.
- `completed` with all stops failed → full or 0?

Pick one and add to [[Driver Rate]]. Ties to ADR-019 (duration-based salary already handles partial trips correctly; per-trip is the loose end).

### Rounding rule for allocation

**Problem.** `OrderLocation.allocated_share` from [[Allocation Rule]] divides revenue across N drops. With currency that has 2 decimal places, `100 / 3 = 33.33` and the residual cent goes... somewhere.

**Decision needed.** Where the residual lands:

- Last drop absorbs the rounding error.
- First drop absorbs.
- Largest-share drop absorbs.
- Round half-up at each step; let `Σ allocated_share` differ from `revenue` by a few cents.

Pick one and document on [[Allocation Rule]]. Affects all per-stop allocation methods.

### `by_quoted_sub_price` allocation method spec

**Problem.** [[Allocation Rule]] lists `by_quoted_sub_price` as a method with the note "Direct per-stop price from quotation line items — manual setup." No schema for what those line items look like.

**Decision needed.** Either spec the line-item table (per-Customer-Location prices on the [[Quotation Service]]) or remove the method until needed.

### Stop Evidence trust (server time vs. device time)

**Problem.** [[Stop Evidence]] captures timestamps. Driver mobile devices have system clocks the driver can change — a back-dated evidence record is a real risk for delivery disputes and time-window compliance reporting.

**Decision needed.** Authoritative timestamp:

- Server-set on receipt (loses information about the actual capture moment when offline).
- Device-set, server-recorded `received_at` separately, both stored.
- Device-set with a signed timestamp from a network time source the device fetches.

Default leaning: store both, use server `received_at` for compliance, device timestamp for ops display.

### Container / pickup requirements

**Problem.** Some pickups need specific containers, dunnage, or equipment (forklift at the stop, refrigerated trailer for the leg). Not modelled today. Operators tell drivers in notes.

**Decision needed.** Whether to type these requirements (`requires_forklift`, `requires_reefer`) on [[Order Location]] / [[Service Stop]] for VRP filtering and driver pre-trip checks, or keep as free-text notes. Likely a `requirements: array<string>` field with a small controlled vocabulary.

---

## How to use this doc

When picking up the next group, copy the relevant section's "Decision needed" line into the chat to start the conversation. The Considerations bullets are not the answer — they are anchors so we don't restart from zero.

When a decision is locked, the item moves out of here and into:

- An ADR under `03-Design-Decisions/` (the **why**).
- One or more concept-doc / overview / integration edits (the **what**).
- An update to [[Home]] and any affected [[Model — Order to Work Order Chain]] sections.

The progress notes in the project memory file (outside the vault) track which groups are complete.
