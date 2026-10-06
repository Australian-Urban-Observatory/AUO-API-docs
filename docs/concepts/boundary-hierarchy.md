# Boundary hierarchy

Every value the API returns is recorded for one **area**: a geographic unit identified by a
`community_code`, a **boundary type**, and a **year** (an area's code can change between
years at the same boundary type — e.g. suburb boundaries were redrawn between the 2018 and
2021 ASGS releases).

## Boundary types

From coarsest to finest:

```
state → study_region → lga → ssc (suburb) → sa1 → meshblock
```

- **state**, **study_region**, **lga**, **ssc**, **sa1** — the levels most endpoints operate
  on. Pass the boundary type as a string, e.g. `boundary_type=lga`. State, study region and
  LGA are open to everyone; **suburb and SA1 need a key whose access covers them** (see
  [Scope, grants & auth](scope-and-auth.md#the-public-baseline)).
- **meshblock** — the finest level held by the API, used only as the aggregation base for
  AUO-computed indicators (see [Rollup](../reference/rollup.md)). Meshblock values are
  **never returned directly, to anyone**: every value/profile/ranking-style endpoint refuses
  them as out of scope. They only ever surface as part of an aggregate.

Each area at LGA level and below carries denormalised parent identifiers (its state, study
region, LGA, suburb, SA1), which is what makes percentile-within-LGA (`pl`) and
percentile-within-city (`pc`) variants possible — see [Variants & percentile basis](variants.md).

## Census/ABS indicators have no meshblock table

AUO-computed indicators are aggregated from meshblock-level data using
`weighted_mean_person`. Census/ABS indicators and the `housing_*` indicators have **no**
meshblock-level source table at all — their `aggregation_method` is `unsupported`, which
means they cannot be rolled up over a custom-drawn region (`CUSTOM_REGION_NOT_SUPPORTED`).
Check `custom_region_supported` on [indicator metadata](../reference/metadata.md) before
attempting a custom-region rollup.

## Cross-year boundary concordance

Suburb (SSC) boundaries in particular did not survive the 2018→2021 ASGS revision unchanged.
For LGA-level trend queries that span 2018, the API resolves the 2018 leg via a stored
concordance mapping where one exists, and returns an explicit warning where it doesn't —
never a silent interpolation across boundaries that don't actually match. See
[Trend](../reference/trend.md#the-2018-leg) for the exact behaviour.

For SSC- and SA1-level areas, no concordance is defined at all: a trend spanning 2018 at
those levels always returns the 2018 point as `null` with a `CONCORDANCE_NOT_SUPPORTED`
warning.
