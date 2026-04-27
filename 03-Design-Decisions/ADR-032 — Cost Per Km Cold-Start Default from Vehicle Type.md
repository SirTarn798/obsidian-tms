# ADR-032 — Cost Per Km Cold-Start Default from Vehicle Type

#decision #module/cost-ledger #module/work-order #status/active

## Status

Accepted. Locks the cold-start fallback for [[Cost Per Km]]. Promotes per-bucket default rates on [[Vehicle Type]] from v2 to v1.

## Context

[[Cost Per Km]] is computed per `(vehicle, bucket)` from a lookback window of historical [[Cost Event]]s and km driven, per [[ADR-016 — Cost Per Km Buckets]]. A brand-new vehicle has no history. Day-1 trips can't compute `cost_per_km` for the fuel / maintenance / tire buckets — the lookback window is empty.

Three cold-start options were on the table:

- **Vehicle Type seed.** [[Vehicle Type]] carries default per-bucket CPK that new vehicles inherit until they have data of their own.
- **Fleet-wide average.** Use the rolling fleet average across all vehicles for the bucket as the seed.
- **Operator-set seed at vehicle creation.** Manual entry per Vehicle, decays to computed once data arrives.

Vehicle Type is the right level for the default because the same physical class of truck (a 22-wheel) consumes fuel and tires similarly regardless of which one is parked in the yard. Fleet-wide averages would mix 4-wheel pickups and 22-wheel rigs into a meaningless number. Operator-set per-vehicle adds maintenance burden and is fragile when entered carelessly.

## Decision

**Per-bucket default CPK rates live on [[Vehicle Type]]. Computed CPK from the lookback window takes over as soon as the lookback yields any non-zero events and km. Otherwise, the Vehicle Type default is used.**

### Vehicle Type Fields (promoted to v1)

| Field | Type | Description |
|---|---|---|
| `default_fuel_per_km` | decimal | Seed CPK for the `fuel` bucket. Used when computed CPK has no data. |
| `default_maintenance_per_km` | decimal | Same for `maintenance` bucket. |
| `default_tire_per_km` | decimal | Same for `tire` bucket. |

These are operator-maintained — the platform doesn't synthesise them. A reasonable starting value is the previous-system fleet rate; subsequent edits follow the close-and-replace nature of [[Cost Rule]]s only insofar as historical computations are unaffected (defaults are read at query time; changing them affects future cold-start computations only).

Buckets are extensible per [[ADR-016]]; new buckets (e.g. `lubricant`) get a corresponding `default_<bucket>_per_km` field on Vehicle Type.

### Resolution Rule

For each `(vehicle, bucket)` at WO cost-calculation time:

```
events_in_window = CostEvents matching (vehicle, bucket) in lookback window
km_in_window     = km driven by vehicle in lookback window

IF events_in_window non-empty AND km_in_window > 0:
    cpk(vehicle, bucket) = Σ events.amount / km_in_window     (computed)
ELSE:
    cpk(vehicle, bucket) = vehicle.vehicle_type.default_<bucket>_per_km   (seed)
```

No threshold (no "after 500 km" or "after N events") — any computed value displaces the default. The lookback window itself bounds how reactive the rate is; a single early event on day 2 still gives a real signal worth using.

### Sub-Contracted Vehicles

Sub-contracted [[Vehicle]] rows do not compute or read CPK at all per [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]. The Vehicle Type defaults exist for the **type** but are simply not consulted on a sub-contract WO — its cost shape is the `subcontract_payout` event, not the per-km buckets.

### UI Hint

When a vehicle is reading the default rather than computed CPK, dashboards may surface a "seed value, not yet computed" hint so operators know the figure is provisional. Optional polish; not enforced.

## Consequences

- **Good:** Day-1 trips have a usable, defensible cost figure for VRP / quotation comparisons.
- **Good:** Defaults live where the type-level facts already live (Vehicle Type), keeping the data model minimal.
- **Good:** Cutover from default to computed is automatic — no operator action.
- **Good:** Per-bucket granularity matches the existing [[ADR-016]] bucket structure.
- **Tradeoff:** Operators must populate `default_*_per_km` per Vehicle Type at platform setup. Bad defaults yield bad cold-start estimates. Mitigation: surface defaults in the Vehicle Type UI; review during onboarding.
- **Tradeoff:** No blending between default and computed during the early period. A single anomalous early event swings the rate. Acceptable — the lookback window's smoothing kicks in after a few events; until then, signal beats noise.

## Open Items

- **Per-bucket lookback minimums.** If experience shows a single noisy early event drives CPK too far, introduce a minimum `events_in_window` threshold below which the default still applies. Not v1.
- **Vehicle-level override.** Operator-set seed for an unusual individual vehicle (e.g. a damaged unit with known above-fleet maintenance burden). Deferred until needed.

## Related

- [[Vehicle Type]]
- [[Cost Per Km]]
- [[Vehicle]]
- [[Cost Event]]
- [[ADR-016 — Cost Per Km Buckets]]
- [[ADR-029 — Vehicle Pool Unified for In-House and Sub-Contract]]
