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
| `boundary_type` | string | yes | The kind of area forming the distribution |
| `scope` | string | yes | `national`, `state`, `study_region`, or `lga` |
| `scope_code` | string | if scope needs one | The [AUO Area ID](../concepts/areas.md) of the state, study region or LGA — see [Rank → scopes](rank.md#scopes) |
| `area` | string | no | Include this area's own value/position — an AUO Area ID |
| `community_code` | string | no | Alternative to `area`: an ABS code, read **at `boundary_type`** |
| `edition` | string | no | With `community_code`: the ABS edition to look it up in |

Give `area` **or** `community_code` (with an optional `edition`), not both; `edition` without
`community_code` is rejected. Naming neither is fine — `own` is then left out.

There is **no `variant` parameter** on this endpoint — the distribution and any own-area
comparison are always computed on the raw value `r`. This isn't an oversight; percentile
variants have no meaning as an input to a spread statistic.

The areas that form the distribution are those of `boundary_type` that fall in `scope`, using
the same rule as [Rank → scopes](rank.md#scopes): assigned parents, so 2018 has no state,
study-region or LGA distribution (use `national`).

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
    "area": {
      "area_id": "lga-000123",
      "name": "Yarra",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "27350" }]
    },
    "resolved_from": null,
    "value": 9.1,
    "position": "above_q3"
  }
}
```

- `count` is the number of areas that actually contributed a value **within your effective
  scope** (see [Scope, grants & auth](../concepts/scope-and-auth.md)).
- `own` only appears at all if you supplied `area` or `community_code`. It describes the Area
  (`area`, and `resolved_from` as in [Lookup](lookup.md#response-200)) and gives its raw
  `value` in the requested year. Within it, `position` is always
  present — `null` if `own.value` is `null`, otherwise one of `below_q1`, `q1_to_median`,
  `median_to_q3`, `above_q3`. Boundaries are half-open and lower-inclusive: a value exactly
  equal to `q1` lands in `q1_to_median`, not `below_q1` (and equal to `median`/`q3` similarly
  rolls into the bucket above). Note that the own Area does **not** need to be a member of
  the requested `scope` or even of `boundary_type` when you name it by `area` — it's a reference
  point being compared against the distribution, not a required participant in it.

## Errors

0. `422` — `area` together with `community_code`/`edition`, or `edition` without
   `community_code` ([malformed request](../concepts/errors.md#malformed-requests))
1. `SCOPE_NOT_FOUND` (404)
2. `AREA_NOT_FOUND` (404) — only checked if `area` or `community_code` was supplied
3. `INDICATOR_NOT_FOUND` (404)
4. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — you can't see the indicator anywhere
5. `401`/`403` — areas have recorded values but you can see none of them
6. `INSUFFICIENT_DATA` (422) — fewer than 2 areas in scope have a recorded value
7. `401`/`403` — you supplied an own Area and can't see this indicator at it

Areas you can't see are silently left out of the distribution, as long as at least one
remains. Only when *none* remain (step 5), or the reference area itself is hidden (step 7),
is the request refused. `INSUFFICIENT_DATA` counts only the areas you can see, so it can
also occur when your access leaves just one area with a value.
