# Common Tasks

Real questions, translated into concrete calls. Each of these is deceptively simple until
you hit the ambiguity called out underneath it — that's the point of this page.

## "Which suburb is most liveable?"

This is the most-asked question against this data, and it's ambiguous until you pick a
**basis** and a **boundary level**:

```
GET /v1/rankings?indicator=liveability_index&year=2024&boundary_type=ssc
    &scope=national&variant=p&direction=best&limit=1
```

Swap `variant` for `r`, `pl`, or `pc` and you can get a different winner — see
[Variants & percentile basis](concepts/variants.md). Always say which basis you used.

## "Rank every suburb in my LGA, worst to best"

```
GET /v1/rankings?indicator=liveability_index&year=2024&boundary_type=ssc
    &scope=lga&scope_code=20660&direction=worst&limit=1000
```

## "Which component indicator is dragging this suburb's score down?"

```
POST /v1/comparisons/indicators
{ "indicator_keys": ["walkability", "public_transport_access", "social_infrastructure_access"],
  "community_code": "20141", "boundary_type": "ssc", "year": 2024 }
```

See [Compare indicators](reference/compare-indicators.md) — the response includes each
indicator's polarity and units alongside its value, so you can tell "low and bad" apart from
"low and fine" without a second call.

## "How does this SA1 compare with the median for its LGA?"

```
GET /v1/distribution?indicator=dwelling_density&year=2024&boundary_type=sa1
    &scope=lga&scope_code=20660&community_code=20660010101
```

`own.position` tells you where it sits (`below_q1`, `q1_to_median`, `median_to_q3`,
`above_q3`) — see [Distribution](reference/distribution.md).

## "Has this changed since 2018?"

```
GET /v1/areas/20660/trend?indicator=public_transport_access&boundary_type=lga
```

Check every point's `warning` field before trusting the 2018 value — see
[Trend → the 2018 leg](reference/trend.md#the-2018-leg). At SSC/SA1 level, 2018 is *never*
directly comparable.

## "Which SA1s fall inside this precinct boundary, and what's their combined profile?"

Two steps — resolve membership, then aggregate:

```
POST /v1/spatial/contains
{ "polygon": {...}, "boundary_type": "sa1", "year": 2024 }
```

Then create a [custom region](reference/rollup.md#custom-region-rollup) from the same
polygon to get an aggregate value per indicator, or call
[Profile](reference/profile.md) per SA1 if you need the full per-area breakdown rather than
one combined number.

## "Which neighbouring suburbs do better than mine on X?"

```
GET /v1/areas/20141/neighbours?boundary_type=ssc&year=2024
```

Then feed the returned codes into
[Compare areas](reference/compare-areas.md) with your own suburb included.

## "What's the correlation between density and social infrastructure access?"

```
POST /v1/correlations
{ "indicator_a": "dwelling_density", "indicator_b": "social_infrastructure_access",
  "year": 2024, "boundary_type": "ssc", "scope": "study_region", "scope_code": "GMEL" }
```

## "I don't know the indicator key — I just know what I mean"

```
GET /v1/search/indicators?q=how easy it is to walk to shops
```

Check `declined` in the response before trusting `candidates` — a query mentioning cycling
infrastructure, gambling, or crime data gets an explicit decline, not a silent near-miss
match. See [Search](reference/search.md).

## "How exactly is this indicator calculated?"

```
GET /v1/indicators/liveability_index
```

`description`, `source`, and `aggregation_method` are the fields to read for methodology —
see [Indicator metadata](reference/metadata.md).

## Out of scope — the API will decline these

Some things partners commonly ask for aren't in this dataset at all, and the API is designed
to say so rather than approximate:

- Absolute population counts (indicators here are proportions/percentiles, not counts)
- Anything below SA1 (no meshblock- or street-level query surface)
- Projections or forecasts (three observed releases only: 2018, 2021, 2024)
- Causal claims ("did X improve Y") — this is cross-sectional data
- Individual/household-level records — aggregated areal data only
- Travel *times* — only pre-computed 400m/800m access measures

If your integration needs one of these, it needs a different data source, not a different
parameter on this API.
