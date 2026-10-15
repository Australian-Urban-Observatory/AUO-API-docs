# Common Tasks

Real questions, translated into concrete calls. Each of these is deceptively simple until
you hit the ambiguity called out underneath it — that's the point of this page.

!!! note "Places are named by AUO Area ID"
    Examples use real [AUO Area IDs](concepts/areas.md): `lga-000084` is Yarra and `ssc-001205`
    is the suburb Fitzroy (Vic.). Get the ID for any other place from
    [area search](reference/search.md#search-areas) or a
    [coordinate lookup](reference/spatial.md#point-in-polygon). Where an endpoint takes a single
    place you can also send an ABS `community_code` with `boundary_type`.

!!! note "Suburb and SA1 calls need an API key"
    Public access covers non-restricted indicators at state, study-region and LGA level only.
    Every example below that uses `boundary_type=ssc` or `sa1` needs an `Authorization:
    Bearer <key>` header, and only returns the areas your key's access covers. See
    [Scope, grants & auth](concepts/scope-and-auth.md).

## "Which suburb is most liveable?"

This is the most-asked question against this data, and it's ambiguous until you pick a
**basis** and a **boundary level**:

```
GET /v1/rankings?indicator=urban_liveability_index&year=2024&boundary_type=ssc
    &scope=national&variant=p&direction=best&limit=1
```

Swap `variant` for `r`, `pl`, or `pc` and you can get a different winner — see
[Variants & percentile basis](concepts/variants.md). Always say which basis you used. The
ranking covers only the suburbs your access reaches; with no suburb access at all you get a
`401`/`403`, not an empty list. Each ranking entry carries the suburb's `area_id`, `name` and
`context` (its LGA and state).

## "Rank every suburb in my LGA, worst to best"

```
GET /v1/rankings?indicator=urban_liveability_index&year=2024&boundary_type=ssc
    &scope=lga&scope_code=lga-000084&direction=worst&limit=1000
```

`scope_code` is the LGA's AUO Area ID. The suburbs ranked are those the data source assigns to
that LGA in 2024, which is not always every suburb that touches it — see
[Rank → scopes](reference/rank.md#scopes). A scoped ranking for 2018 is empty; use
`scope=national`.

## "Which component indicator is dragging this suburb's score down?"

```
POST /v1/comparisons/indicators
{ "indicator_keys": ["social_infrastructure_community_and_sport", "social_infrastructure_education",
                     "social_infrastructure_health_services"],
  "area": "ssc-001205", "year": 2024 }
```

See [Compare indicators](reference/compare-indicators.md) — the response includes each
indicator's polarity and units alongside its value, so you can tell "low and bad" apart from
"low and fine" without a second call.

## "How does this SA1 compare with the median for its LGA?"

```
GET /v1/distribution?indicator=urban_liveability_index&year=2024&boundary_type=sa1
    &scope=lga&scope_code=lga-000084&area=sa1-014610
```

`own.position` tells you where it sits (`below_q1`, `q1_to_median`, `median_to_q3`,
`above_q3`) — see [Distribution](reference/distribution.md).

## "Has this changed since 2018?"

```
GET /v1/areas/trend?area=lga-000084&indicator=transport_percent_dwellings_400m_regular_pt
```

`points` has an entry for each release the area exists in, so check `unavailable_years` for
releases it is missing from and `predecessors` for areas that merged into it — see
[Trend](reference/trend.md). Suburbs and SA1s have 2018 points too when the area exists in
2018, but a suburb that split or merged between ABS editions is a different Area with its own
history.

## "Which SA1s fall inside this precinct boundary, and what's their combined profile?"

Two steps — resolve membership, then aggregate:

```
POST /v1/spatial/contains
{ "polygon": {...}, "boundary_type": "sa1", "year": 2024 }
```

Then create a [custom region](reference/rollup.md#custom-region-rollup) from the same
polygon to get an aggregate value per indicator, or call
[Profile](reference/profile.md) per SA1 (by the `area_id`s the first call returned) if you
need the full per-area breakdown rather than one combined number. Custom regions are built
from meshblocks, so they need access that covers meshblocks for the indicator, and a region
must be large enough to be rolled up — see
[Rollup → minimum size](reference/rollup.md#minimum-size).

## "Which neighbouring suburbs do better than mine on X?"

```
GET /v1/areas/neighbours?area=ssc-001205&year=2024
```

Then feed the returned `area_id`s into
[Compare areas](reference/compare-areas.md) with your own suburb included.

## "What's the correlation between density and social infrastructure access?"

```
POST /v1/correlations
{ "indicator_a": "walkability_dwelling_density", "indicator_b": "social_infrastructure_index",
  "year": 2021, "boundary_type": "lga", "scope": "study_region",
  "scope_code": "study_region-000014" }
```

## "I only know the place name — what's its ID?"

```
GET /v1/search/areas?q=brunswick
```

Candidates come back with `area_id`, `boundary_type`, `source_codes`, and `context` (the LGA
and state) so you can tell two same-named places apart. Suburbs only appear if your access
covers them. See [Search → areas](reference/search.md#search-areas).

## "I have an ABS code from my own data — what Area is it?"

```
GET /v1/values?community_code=20924&boundary_type=ssc&indicator=urban_liveability_index&year=2024
```

The response's `area.area_id` is the permanent ID, and `resolved_from` says which ABS edition
the code matched. Without `edition`, a code is read in the newest edition that contains it —
so for suburb codes, which differ between the 2016 and 2021 editions, say which you mean with
`&edition=SSC%202016` (Fitzroy is `20918` there) or `&edition=SAL%202021`. See [Areas](concepts/areas.md#naming-an-area-in-a-request).

## "I don't know the indicator key — I just know what I mean"

```
GET /v1/search/indicators?q=how easy it is to walk to shops
```

Check `declined` in the response before trusting `candidates` — a query mentioning cycling
infrastructure, gambling, or crime data gets an explicit decline, not a silent near-miss
match. See [Search](reference/search.md).

## "How exactly is this indicator calculated?"

```
GET /v1/indicators/urban_liveability_index
```

`description`, `source`, and `aggregation_method` are the fields to read for methodology —
see [Indicator metadata](reference/metadata.md).

## Out of scope — the API will decline these

Some things partners commonly ask for aren't in this dataset at all, and the API is designed
to say so rather than approximate:

- Absolute population counts (indicators here are proportions/percentiles, not counts)
- Anything below SA1 (meshblock values are never returned directly; they only feed
  [custom-region and area-group roll-ups](reference/rollup.md))
- Projections or forecasts (three observed releases only: 2018, 2021, 2024)
- Causal claims ("did X improve Y") — this is cross-sectional data
- Individual/household-level records — aggregated areal data only
- Travel *times* — only pre-computed 400m/800m access measures

If your integration needs one of these, it needs a different data source, not a different
parameter on this API.
