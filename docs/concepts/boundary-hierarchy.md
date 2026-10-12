# Boundary hierarchy

Every value the API returns is recorded for one **area**: a geographic unit with a permanent
[AUO Area ID](areas.md), a **boundary type**, and a value for each **year** it exists in. An
area's ABS code is not its identity — the same code can mean a different place in another ABS
edition (suburb boundaries were renumbered between the 2016 and 2021 editions) — so the API
identifies areas by AUO Area ID. See [Areas & AUO Area IDs](areas.md).

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

Each area at LGA level and below has **assigned parents** — the state, study region, LGA (and, for
SA1s, suburb) the data source files it under, in each year. These assignments are what makes
percentile-within-LGA (`pl`) and percentile-within-city (`pc`) variants possible and what
`scope` means on rankings — see [Variants & percentile basis](variants.md) and
[Areas → assigned parents](areas.md#assigned-parents). They are the source's own assignment,
not geometric containment.

## Census/ABS indicators have no meshblock table

AUO-computed indicators are aggregated from meshblock-level data using
`weighted_mean_person`. Census/ABS indicators and the `housing_*` indicators have **no**
meshblock-level source table at all — their `aggregation_method` is `unsupported`, which
means they cannot be rolled up over a custom-drawn region (`CUSTOM_REGION_NOT_SUPPORTED`).
Check `custom_region_supported` on [indicator metadata](../reference/metadata.md) before
attempting a custom-region rollup.

## Comparing across years

Because an Area is the same place across releases, a [trend](../reference/trend.md) is simply
one Area's values in each year it exists in. There is no cross-year mapping to apply and
nothing is interpolated:

- LGAs, SA1s, states and study regions keep the same Area wherever their ABS code is
  unchanged.
- Suburbs are matched across the 2016 and 2021 ABS editions by boundary overlap, so most
  suburbs have a 2018, 2021 and 2024 point. Suburbs that split, merged or moved study region
  don't match and become separate Areas.
- Where two Areas merged into one (Rockdale and Botany Bay into Bayside (NSW)), the later Area
  lists the earlier ones as `predecessors` but never absorbs their values.
- A year in which an Area has no feature has no point, and the trend says why
  (`AREA_NOT_IN_YEAR`).
