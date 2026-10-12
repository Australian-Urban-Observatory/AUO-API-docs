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

```json
{
  "indicator": "walkability",
  "year": 2021,
  "variant": "r",
  "scope": "national",
  "direction": "best",
  "total": 542,
  "areas": [
    { "area_id": "lga-000123", "value": 9.1, "rank": 1 },
    { "area_id": "lga-000044", "value": 9.1, "rank": 1 },
    { "area_id": "lga-000310", "value": 8.9, "rank": 3 }
  ]
}
```

Each entry is the Area's `area_id`, its `value` and its `rank`. To get names and source
codes for the entries, look the Areas up by `area_id` (for example with [Lookup](lookup.md) or
[Compare areas](compare-areas.md)).

- `total` is the count of areas that had a recorded value and were within **your** effective
  scope — not the size of the scope you requested before filtering.
- Equal values share the same `rank` (see the tie at rank 1 above); the next distinct value
  resumes at the count of areas ahead of it, not `rank + 1`.
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
