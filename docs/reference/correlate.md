# Correlate

`correlate` — the Pearson correlation coefficient between two indicators, across every area
in a scope with a recorded raw value for both. A **computed resource**, same lifecycle as
[Compare areas](compare-areas.md).

## Create

```
POST /v1/correlations
```

```json
{
  "indicator_a": "dwelling_density",
  "indicator_b": "social_infrastructure_access",
  "year": 2021,
  "boundary_type": "ssc",
  "scope": "study_region",
  "scope_code": "GMEL"
}
```

No `variant` field — always computed on raw (`r`) values, like [Distribution](distribution.md).

### Response — 201

```json
{
  "id": 8,
  "url": "/v1/correlations/8",
  "result": {
    "indicator_a": "dwelling_density",
    "indicator_b": "social_infrastructure_access",
    "year": 2021,
    "scope": "study_region",
    "count": 214,
    "coefficient": -0.42
  }
}
```

`count` is the number of areas that had a recorded raw value for **both** indicators, within
your effective scope.

### Errors (create)

1. `SCOPE_NOT_FOUND` (404)
2. `DUPLICATE_INDICATOR` (422) — `indicator_a` and `indicator_b` are the same key
3. `INDICATOR_NOT_FOUND` (404) — names every key that failed to resolve
4. `INSUFFICIENT_DATA` (422) — fewer than 2 areas have both values, checked only after
   scope/indicators all resolved successfully

No `OUT_OF_SCOPE` at creation — same reasoning as area/indicator comparisons.

## Retrieve

```
GET /v1/correlations/{correlation_id}
```

Live recompute against the caller's own scope. Supports `Accept: text/csv`.

### Errors (retrieve)

1. `RESOURCE_NOT_FOUND` (404)
2. `OUT_OF_SCOPE` (403) — the scope-implied candidate area set is entirely invisible to you
3. `INSUFFICIENT_DATA` (422) — at least one candidate area is visible, but fewer than two
   have paired data (a distinct outcome from `OUT_OF_SCOPE` — some data is visible, just not
   enough of it)
