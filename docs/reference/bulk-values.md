# Bulk values

`bulk-values` — the value of one indicator for **every** area of one boundary type in a scope,
for one year, in a single response. Use it to paint a map or fill a table; use
[Rank areas](rank.md) when you want an ordered top or bottom N.

```
GET /v1/values/bulk
```

## Query parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `indicator` | string | yes | Indicator key |
| `year` | integer | yes | |
| `boundary_type` | string | yes | The kind of area to return, e.g. `lga`, `ssc`, `sa1` |
| `scope` | string | yes | `national`, `state`, `study_region`, or `lga` |
| `scope_code` | string | if `scope` is `state`/`study_region`/`lga` | The [AUO Area ID](../concepts/areas.md) of the state, study region or LGA. Scopes work exactly as on [Rank areas → scopes](rank.md#scopes) |
| `variants` | string | no | Comma-separated list from `r`, `p`, `pl`, `pc`. Default: all four. Repeats are dropped and the order you give is kept. See [Variants](../concepts/variants.md) |

Unlike [Rank areas](rank.md), there is no `direction` and no `limit`, and the variants are a
list (`variants`) rather than a single `variant`.

## Response — 200

`GET /v1/values/bulk?indicator=urban_liveability_index&year=2018&boundary_type=ssc&scope=national`

The response below is trimmed to three of the 3,094 areas; the values are as returned.

```json
{
  "indicator": "urban_liveability_index",
  "year": 2018,
  "boundary_type": "ssc",
  "scope": "national",
  "scope_code": null,
  "variants": ["r", "p", "pl", "pc"],
  "total": 3094,
  "areas": [
    {
      "area_id": "ssc-000001",
      "name": "Abbotsbury",
      "context": "Fairfield, New South Wales",
      "source_codes": [{ "edition": "SAL 2021", "code": "10002" }, { "edition": "SSC 2016", "code": "10002" }],
      "values": { "r": 98.7, "p": 56.0, "pl": 48.0, "pc": 47.0 }
    },
    {
      "area_id": "ssc-000619",
      "name": "Mosman",
      "context": "Mosman, New South Wales",
      "source_codes": [{ "edition": "SAL 2021", "code": "12749" }, { "edition": "SSC 2016", "code": "12738" }],
      "values": { "r": 103.7, "p": 93.0, "pl": null, "pc": 91.0 }
    },
    {
      "area_id": "ssc-003225",
      "name": "Yarralumla",
      "context": "Unincorporated ACT, Australian Capital Territory",
      "source_codes": [{ "edition": "SAL 2021", "code": "80136" }, { "edition": "SSC 2016", "code": "80131" }],
      "values": { "r": 103.0, "p": 91.0, "pl": 79.0, "pc": 79.0 }
    }
  ]
}
```

- `indicator`, `year`, `boundary_type`, `scope` and `scope_code` echo the request;
  `scope_code` is `null` for `scope=national`.
- `variants` is the list actually returned: the order you asked for, without repeats. Each
  area's `values` has exactly these keys.
- Each entry describes the Area (`area_id`, `name`, `context`, `source_codes`) in the same way
  as [Rank areas](rank.md#response-200), and `source_codes` is how you join the values to a map
  layer or table keyed by ABS code. There is no `boundary_type` per entry (it is the one
  requested) and no `rank`.
- An area is listed if **at least one** requested variant has a recorded value there. A
  variant with no value for that area is `null` — Mosman above has no `pl` value. See
  [Variants → null values](../concepts/variants.md#null-values).
- Areas are ordered by `area_id` (plain bytewise string order, so `ssc-000619` comes before
  `ssc-003225`). Don't read the order as a ranking.
- `total` is the number of areas returned (the length of `areas`), counting only areas within
  **your** effective scope — see [Auth](#auth).
- There is no paging and no 1000-row cap: you get every area in one response. Only values are
  returned, no geometry.
- Nothing has a recorded value at all in a scope you are allowed to see → `total: 0`,
  `areas: []` — a valid `200`, not an error.

### Choosing variants

Ask only for the variants you need. With `variants=p,pc` the same kind of request for the
suburbs of one study region (Melbourne, `study_region-000014`, 2021) returns 445 areas, each with
just those two values:

`GET /v1/values/bulk?indicator=urban_liveability_index&year=2021&boundary_type=ssc&scope=study_region&scope_code=study_region-000014&variants=p,pc`

```json
{
  "indicator": "urban_liveability_index",
  "year": 2021,
  "boundary_type": "ssc",
  "scope": "study_region",
  "scope_code": "study_region-000014",
  "variants": ["p", "pc"],
  "total": 445,
  "areas": [
    {
      "area_id": "ssc-001008",
      "name": "Abbotsford (Vic.)",
      "context": "Yarra, Victoria",
      "source_codes": [{ "edition": "SAL 2021", "code": "20002" }, { "edition": "SSC 2016", "code": "20002" }],
      "values": { "p": 98.0, "pc": 98.0 }
    }
  ]
}
```

(trimmed to the first of the 445 areas.)

## Scopes

Scopes behave exactly as they do for [Rank areas](rank.md#scopes): the areas returned are those
that have the scope Area as their **assigned parent** in the requested year, not those that
geometrically sit inside it. A `scope_code` that is not the AUO Area ID of an Area of the kind
`scope` names (or is missing for `state`/`study_region`/`lga`) is `404 SCOPE_NOT_FOUND`.

!!! warning "2018: use `scope=national`"
    2018 areas have no assigned parents, so for 2018 only `scope=national` returns anything.
    `state`, `study_region` and `lga` scopes answer `200` with `total: 0`. To fetch every 2018
    suburb in one call, ask for the whole country:

    `GET /v1/values/bulk?indicator=urban_liveability_index&year=2018&boundary_type=ssc&scope=national`

    (3,094 suburbs for that indicator, all four variants, in the example above.)

## Response size

There is no row cap, so the response is as large as the scope. Responses are not compressed
by the API. For `urban_liveability_index`, suburbs for all of Australia in 2018 (3,094 areas, all
four variants) came to about 0.7 MB, while SA1s for all of Australia (about 40,000 areas) came
to about 9 MB, whichever of the 2018, 2021 or 2024 releases and whether or not `variants`
was narrowed to one. If you don't need all four variants, pass `variants=`; if you don't need the
whole country, use a narrower `scope`.

A scope that holds more than **100,000** areas with a value is refused with
`422 SCOPE_TOO_LARGE` and a message asking you to narrow the scope. Current national SA1
scopes are well inside this limit.

## Auth

Access is the same as [Rank areas → Auth](rank.md#auth). Without a key you get the public
baseline: values of non-restricted indicators at LGA level or coarser. With a key, you get the
areas your key's access covers for that indicator. Areas you can't see are silently left out
and `total` counts only what you can see.

Two cases are refused instead of filtered:

- you can't see the indicator at any area, or
- areas *do* have recorded values, but you can see none of them (for example, asking for
  suburbs with no suburb-level access) — a `401`/`403`, **not** an empty `200`.

Meshblock values are never returned. `401` is `AUTHENTICATION_REQUIRED` (no credential sent);
`403` is `OUT_OF_SCOPE`. See [Scope, grants & auth](../concepts/scope-and-auth.md).

## Errors

Every error uses the [standard envelope](../concepts/errors.md#the-error-envelope).

- `REQUEST_INVALID` (422) — a required parameter is missing or mistyped, or `variants` is empty
  or names something other than `r`, `p`, `pl`, `pc`. For a bad `variants` the `details` name
  the offending entries:

    `GET /v1/values/bulk?indicator=urban_liveability_index&year=2018&boundary_type=ssc&scope=national&variants=p,x`

    ```json
    {
      "error": {
        "code": "REQUEST_INVALID",
        "message": "'variants' must be a comma-separated list of 'r', 'p', 'pl', 'pc'; not supported: 'x'.",
        "details": {
          "variants": "p,x",
          "unknown_variants": ["x"],
          "valid_variants": ["r", "p", "pl", "pc"]
        }
      }
    }
    ```

- `UNKNOWN_BOUNDARY_TYPE` (400) — `boundary_type` isn't one of the valid types; `details.valid_boundary_types`
  lists them (`state`, `study_region`, `lga`, `ssc`, `sa1`).
- `SCOPE_NOT_FOUND` (404) — `scope_code` isn't the AUO Area ID of an Area of the kind `scope` names,
  or is missing for a `state`/`study_region`/`lga` scope.
- `INDICATOR_NOT_FOUND` (404) — the indicator key doesn't exist.
- `INSUFFICIENT_DATA` (422) — nothing is stored at all for that boundary type.
- `SCOPE_TOO_LARGE` (422) — the scope holds more than 100,000 areas with a value; narrow the scope.
- `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — the indicator isn't visible to you anywhere,
  or filtering left nothing (see [Auth](#auth)).

Also the usual [version-prefix errors](../getting-started.md#versioning). See
[Errors & warnings](../concepts/errors.md) for the shared codes.
