# Distribution

`distribution` — where an area sits in the spread of its peers, rather than a single
unqualified number.

```
GET /v1/distribution
```

## Query parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `indicator` | string | yes | |
| `year` | integer | yes | |
| `boundary_type` | string | yes | |
| `scope` | string | yes | `national`, `state`, `study_region`, or `lga` |
| `scope_code` | string | if scope needs one | |
| `community_code` | string | no | Include this area's own value/position |

There is **no `variant` parameter** on this endpoint — the distribution and any own-area
comparison are always computed on the raw value `r`. This isn't an oversight; percentile
variants have no meaning as an input to a spread statistic.

## Response — 200

```json
{
  "indicator": "walkability",
  "year": 2021,
  "boundary_type": "lga",
  "scope": "national",
  "min": 2.1,
  "max": 9.4,
  "median": 5.6,
  "q1": 4.2,
  "q3": 7.1,
  "count": 542,
  "own": {
    "community_code": "20660",
    "value": 9.1,
    "position": "above_q3"
  }
}
```

- `count` is the number of areas that actually contributed a value **within your effective
  scope** — filtering, not rejection (see [Scope, grants & auth](../concepts/scope-and-auth.md)).
- `own` only appears at all if you supplied `community_code`. Within it, `position` is always
  present — `null` if `own.value` is `null`, otherwise one of `below_q1`, `q1_to_median`,
  `median_to_q3`, `above_q3`. Boundaries are half-open and lower-inclusive: a value exactly
  equal to `q1` lands in `q1_to_median`, not `below_q1` (and equal to `median`/`q3` similarly
  rolls into the bucket above). Note that `community_code` does **not** need to be a member
  of the requested `scope` — it's a reference point being compared against the distribution,
  not a required participant in it.

## Errors

1. `SCOPE_NOT_FOUND` (404)
2. `AREA_NOT_FOUND` (404) — only checked if `community_code` was supplied
3. `INDICATOR_NOT_FOUND` (404)
4. `INSUFFICIENT_DATA` (422) — fewer than 2 areas in scope have a recorded value

No `OUT_OF_SCOPE` on this endpoint — scope enforcement is filtering only.
