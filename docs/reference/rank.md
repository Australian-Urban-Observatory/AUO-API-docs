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
| `boundary_type` | string | yes | The kind of area to rank, e.g. `lga`, `ssc`, `sa1` |
| `scope` | string | yes | `national`, `state`, `study_region`, or `lga` |
| `scope_code` | string | if `scope` is `state`/`study_region`/`lga` | The [AUO Area ID](../concepts/areas.md) of the state, study region or LGA — see [Scopes](#scopes) |
| `variant` | string | no | Default `r` |
| `direction` | string | no | `best` or `worst` (default `best`) |
| `limit` | integer | no | 1–1000; default **100** |

## Scopes

`scope` narrows which areas are ranked. For `state`, `study_region` and `lga`, `scope_code` is
the AUO Area ID of an Area of **that** kind — an LGA's ID for `scope=lga`, and so on. An ABS
code, a name, or the ID of an Area of a different kind is `404 SCOPE_NOT_FOUND`. (Find an ID
with [area search](search.md#search-areas).)

The ranked areas are those that have the scope Area as their **assigned parent** in the
requested year — the data source's own assignment, not geometric containment (see [Areas →
assigned parents](../concepts/areas.md#assigned-parents)). So "the SA1s of this LGA" is the set
the source computed its LGA percentiles against, an SA1 spanning two LGAs is ranked in just
one, and **2018 areas have no assigned parents, so a `state`/`study_region`/`lga` scoped ranking
for 2018 is empty** (`total: 0`); use `national` for 2018.

A scope Area that no longer appears in the latest release still counts as a known scope: it
simply ranks nothing, rather than being reported as not found.

## Response — 200

`GET /v1/rankings?indicator=walkability_index&year=2021&boundary_type=lga&scope=national&variant=p&direction=best&limit=4`

```json
{
  "indicator": "walkability_index",
  "year": 2021,
  "variant": "p",
  "scope": "national",
  "direction": "best",
  "total": 171,
  "areas": [
    {
      "area_id": "lga-000067",
      "name": "Melbourne",
      "context": "Victoria",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "24600" }, { "edition": "LGA 2016", "code": "24600" }],
      "value": 100.0,
      "rank": 1
    },
    {
      "area_id": "lga-000038",
      "name": "Sydney",
      "context": "New South Wales",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "17200" }, { "edition": "LGA 2016", "code": "17200" }],
      "value": 99.0,
      "rank": 2
    },
    {
      "area_id": "lga-000084",
      "name": "Yarra",
      "context": "Victoria",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "27350" }, { "edition": "LGA 2016", "code": "27350" }],
      "value": 99.0,
      "rank": 2
    },
    {
      "area_id": "lga-000076",
      "name": "Port Phillip",
      "context": "Victoria",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "25900" }, { "edition": "LGA 2016", "code": "25900" }],
      "value": 98.0,
      "rank": 4
    }
  ]
}
```

Each entry describes the Area (`area_id`, `name`, `context`, `boundary_type`, `source_codes`) plus
its `value` and its `rank`. `context` locates the Area for a human, as in
[area search](search.md#search-areas): the state for an LGA, the LGA and state for a suburb, and
`null` for a state or study region.

- `total` is the count of areas that had a recorded value and were within **your** effective
  scope — not the size of the scope you requested before filtering.
- Equal values share the same `rank` (Sydney and Yarra are both rank 2 above); the next distinct
  value is one more than the count of areas ahead of it, so it skips ranks (Port Phillip is rank
  4 — three areas are ahead of it — not rank 3).
- For the raw variant `r`, ranking direction already accounts for the indicator's polarity —
  `direction=best` always means "more liveable," regardless of whether a higher or lower raw
  number is better for that particular indicator (`polarity` `ASC` or `DESC` on the
  [indicator's metadata](metadata.md)). Percentile variants already have this built in, so
  100 is always the best.
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
2. `SCOPE_NOT_FOUND` (404) — `scope_code` isn't the AUO Area ID of an Area of the kind `scope`
   names (or `scope_code` is missing for a `state`/`study_region`/`lga` scope)
3. `INDICATOR_NOT_FOUND` (404)
4. `VARIANT_NOT_SUPPORTED` (422)
5. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — indicator not visible to you anywhere,
   or filtering left nothing (see Auth above)
