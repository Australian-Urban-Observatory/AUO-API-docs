# Trend

`trend` — one indicator, one area, across the releases (2018, 2021, 2024) in which that area
exists.

```
GET /v1/areas/trend
```

## Parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `area` | query, string | one of `area` / `community_code` + `boundary_type` | The [AUO Area ID](../concepts/areas.md) |
| `community_code` | query, string | with `boundary_type`, instead of `area` | An ABS code, resolved to an Area |
| `boundary_type` | query, string | with `community_code` | |
| `edition` | query, string | no | With `community_code`: the ABS edition to look the code up in |
| `indicator` | query, string | yes | |
| `variant` | query, string | no | Default `r` |

The Area is no longer part of the URL path. If you start from a code, the code is resolved
to an Area (see [Naming an Area in a
request](../concepts/areas.md#naming-an-area-in-a-request)) and the trend follows **that Area**
across years — not the code, which can mean a different place in another ABS edition.

## Response — 200

`GET /v1/areas/trend?community_code=20924&boundary_type=ssc&indicator=urban_liveability_index`

Named by its SAL 2021 code, so `resolved_from` records which code matched. Naming the same suburb by `area=ssc-001205` returns the same body with `resolved_from: null`.

```json
{
  "area": {
    "area_id": "ssc-001205",
    "name": "Fitzroy (Vic.)",
    "boundary_type": "ssc",
    "source_codes": [{ "edition": "SAL 2021", "code": "20924" }, { "edition": "SSC 2016", "code": "20918" }]
  },
  "resolved_from": { "edition": "SAL 2021", "code": "20924" },
  "indicator": "urban_liveability_index",
  "variant": "r",
  "points": [
    { "year": 2018, "value": 108.5 },
    { "year": 2021, "value": 108.1 },
    { "year": 2024, "value": 108.0 }
  ],
  "unavailable_years": [],
  "predecessors": []
}
```

- `points` has one entry for each release in which the Area **has a feature**, oldest
  first. A release where the Area exists but has no recorded value is `value: null`.
- `unavailable_years` lists the releases in which the Area has no feature at all, each with
  the reason `AREA_NOT_IN_YEAR`. Those years have no entry in `points`. See
  [Years the Area isn't in](#years-the-area-isnt-in).
- `predecessors` lists Areas that merged into this one — see [Predecessors](#predecessors).
- `area` and `resolved_from` are as in [Lookup](lookup.md#response-200).

`points` is therefore **not** always three entries: a trend can have one, two or three,
depending on how many releases the Area exists in. Read `points` and `unavailable_years`
together.

!!! note "Changed from earlier versions"
    The `community_code_used` field is gone (an Area has one identity across years, so there
    is no per-year code to report), and so are the 2018 `warning`s
    `BOUNDARY_CONCORDANCE_EXCEPTION` and `CONCORDANCE_NOT_SUPPORTED`. 2018 points for suburbs
    and SA1s are no longer refused: they appear whenever the Area exists in 2018.

## Years the Area isn't in

Some Areas are not in every release — a suburb that was created, split or merged between
releases, or an SA1 that only exists in some editions. For a year with no feature the trend
has no point and says why. This is the `unavailable_years` of
`GET /v1/areas/trend?area=lga-000172&indicator=urban_liveability_index` (Bayside (NSW), which
does not exist in 2018):

```json
"unavailable_years": [
  {
    "year": 2018,
    "reason": {
      "code": "AREA_NOT_IN_YEAR",
      "message": "This Area has no feature in 2018, so it has no point for that Year.",
      "details": { "area": "lga-000172", "year": 2018 }
    }
  }
]
```

The value for that year is never guessed from a neighbouring or older Area. The response is
still `200`.

## Predecessors

When an Area was formed by a merge, `predecessors` lists the Areas it was formed from, each
described like any Area (`area_id`, `name`, `boundary_type`, `source_codes`), ordered by name:

```json
"predecessors": [
  {
    "area_id": "lga-000004",
    "name": "Botany Bay",
    "boundary_type": "lga",
    "source_codes": [{ "edition": "LGA 2016", "code": "11100" }]
  },
  {
    "area_id": "lga-000033",
    "name": "Rockdale",
    "boundary_type": "lga",
    "source_codes": [{ "edition": "LGA 2016", "code": "16650" }]
  }
]
```

The later Area gets **no 2018 value** of its own, and the predecessors' values are never
summed or averaged into it. To see the earlier history, request each predecessor's own trend
by its `area_id`. Examples of merges: Rockdale (`lga-000033`) and Botany Bay (`lga-000004`) into Bayside (NSW)
(`lga-000172`), shown above; Mallala (`lga-000110`) into Adelaide Plains (`lga-000174`).

## Auth

Rejecting, not filtering — same as [Lookup](lookup.md): naming an indicator or area your
scope doesn't cover gets `401 AUTHENTICATION_REQUIRED` (no credential) or `403
OUT_OF_SCOPE` (credential presented). Predecessors are listed whatever your scope; their
values are not returned, and reading them is subject to the usual access checks.

## Errors

0. `422` — the Area is named both ways, or neither way ([malformed
   request](../concepts/errors.md#malformed-requests))
1. `AREA_NOT_FOUND` (404)
2. `INDICATOR_NOT_FOUND` (404)
3. `401`/`403` — you can't see this indicator at any area
4. `401`/`403` — you can see the indicator elsewhere, but not at this area
5. `VARIANT_NOT_SUPPORTED` (422)
