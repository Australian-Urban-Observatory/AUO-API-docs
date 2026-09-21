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
- No areas matching → `total: 0`, `areas: []` — this is a valid `200`, not an error.

## Auth

Public. Areas outside your effective scope are silently excluded from the candidate set
(filtering, not rejection) — `total` reflects the filtered set.

## Errors

1. `SCOPE_NOT_FOUND` (404) — `scope_code` doesn't resolve for the given `scope`
2. `INDICATOR_NOT_FOUND` (404)
3. `VARIANT_NOT_SUPPORTED` (422)
4. `INVALID_LIMIT` (422) — checked before anything else, since it needs no database access
