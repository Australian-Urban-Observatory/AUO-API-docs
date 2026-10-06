# Rank areas

`rank` — the top or bottom N areas within a scope, for one indicator/year/variant.

```
GET /v1/rankings
```

## Query parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `indicator` | string | yes | |
| `year` | integer | yes | |
| `boundary_type` | string | yes | |
| `scope` | string | yes | `national`, `state`, `study_region`, or `lga` |
| `scope_code` | string | if `scope` is `state`/`study_region`/`lga` | The specific state/region/LGA code |
| `variant` | string | no | Default `r` |
| `direction` | string | no | `best` or `worst` (default `best`) |
| `limit` | integer | no | 1–1000; default **100** |

## Response — 200

```json
{
  "indicator": "walkability",
  "year": 2021,
  "variant": "r",
  "scope": "national",
  "direction": "best",
  "total": 542,
  "areas": [
    { "community_code": "20660", "value": 9.1, "rank": 1 },
    { "community_code": "20140", "value": 9.1, "rank": 1 },
    { "community_code": "20310", "value": 8.9, "rank": 3 }
  ]
}
```

- `total` is the count of areas that had a recorded value and were within **your** effective
  scope — not the size of the scope you requested before filtering.
- Equal values share the same `rank` (see the tie at rank 1 above); the next distinct value
  resumes at the count of areas ahead of it, not `rank + 1`.
- For the raw variant `r`, ranking direction already accounts for the indicator's polarity —
  `direction=best` always means "more liveable," regardless of whether a higher or lower raw
  number is better for that particular indicator.
- No area has a recorded value at all → `total: 0`, `areas: []` — a valid `200`, not an
  error.

## Auth

Candidates are limited to areas where your effective scope covers this indicator, so a
public request only ever ranks state, study-region or LGA areas of a non-restricted
indicator. Areas you can't see are silently excluded — `total` reflects the filtered set.

Two cases are refused instead of filtered:

- you can't see the indicator at any area, or
- areas *do* have recorded values, but you can see none of them (for example, ranking
  suburbs with no suburb-level access) — a `401`/`403`, **not** an empty `200`.

So `total: 0` means there's genuinely no data, not that your access was too narrow.

## Errors

1. `INVALID_LIMIT` (422) — checked before anything else, since it needs no database access
2. `SCOPE_NOT_FOUND` (404) — `scope_code` doesn't resolve for the given `scope`
3. `INDICATOR_NOT_FOUND` (404)
4. `VARIANT_NOT_SUPPORTED` (422)
5. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — indicator not visible to you anywhere,
   or filtering left nothing (see Auth above)
