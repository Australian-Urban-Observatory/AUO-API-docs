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
4. `401`/`403` — indicator access, as above
5. `INSUFFICIENT_DATA` (422) — fewer than 2 areas you can see have both values

Scope is applied to you at creation, unlike naming individual codes in a comparison:

- `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — you can't see one of the two
  indicators at any area (checked after both resolve), or areas have values for both but you
  can see none of them. This comes before `INSUFFICIENT_DATA`, so you never get "not enough
  data" for a correlation you aren't allowed to see at all.

## Retrieve

```
GET /v1/correlations/{correlation_id}
```

Live recompute against the caller's own scope, returning `{indicator_a, indicator_b, year,
scope, count, coefficient}` (no `id`/`url`/`result` wrapper). Supports `Accept: text/csv`.

### Errors (retrieve)

1. `RESOURCE_NOT_FOUND` (404)
2. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — the scope-implied candidate area set
   is entirely invisible to you
3. `INSUFFICIENT_DATA` (422) — at least one candidate area is visible, but fewer than two
   have paired data (a distinct outcome from `OUT_OF_SCOPE` — some data is visible, just not
   enough of it)
