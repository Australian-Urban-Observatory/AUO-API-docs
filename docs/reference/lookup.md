# Lookup a value

`lookup` — one indicator, one area, one year, one variant.

```
GET /v1/values
```

## Query parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `area` | string | one of `area` / `community_code` + `boundary_type` | The [AUO Area ID](../concepts/areas.md) |
| `community_code` | string | with `boundary_type`, instead of `area` | An ABS code, resolved to an Area |
| `boundary_type` | string | with `community_code` | e.g. `lga`, `ssc`, `sa1` |
| `edition` | string | no | With `community_code`: look the code up in this ABS edition (e.g. `SAL 2021`) rather than the newest one containing it |
| `indicator` | string | yes | Indicator key |
| `year` | integer | yes | |
| `variant` | string | no | Default `r`. See [Variants](../concepts/variants.md) |

Name the Area one way only — see
[Naming an Area in a request](../concepts/areas.md#naming-an-area-in-a-request).

## Response — 200

`GET /v1/values?area=lga-000084&indicator=walkability_index&year=2021`

```json
{
  "area": {
    "area_id": "lga-000084",
    "name": "Yarra",
    "boundary_type": "lga",
    "source_codes": [{ "edition": "LGA 2021", "code": "27350" }, { "edition": "LGA 2016", "code": "27350" }]
  },
  "resolved_from": null,
  "indicator": "walkability_index",
  "year": 2021,
  "variant": "r",
  "value": 7.3,
  "polarity": "ASC",
  "units": ""
}
```

- `area` describes the Area the request landed on.
- `resolved_from` is the one source code (`{edition, code}`) a `community_code` request
  matched, and `null` when you sent `area`. Look here to see which ABS edition a bare code
  resolved in.
- `value` is `null` if no value is recorded for this exact combination — see
  [Variants → null values](../concepts/variants.md#null-values). A year in which the Area has
  no data simply comes back `null` here; use [Trend](trend.md) if you want to tell "no feature
  that year" apart from "no value".
- `polarity` and `units` come from the indicator's own metadata and are always included.
  `polarity` is `ASC` where a higher raw value is better and `DESC` where a lower one is — see
  [metadata](metadata.md).

## Auth

Public only for a non-restricted indicator at state, study-region or LGA level. Suburb and
SA1 values, and partner-restricted indicators anywhere, need a key whose access covers that
exact indicator at that exact area. Without a key the request gets `401
AUTHENTICATION_REQUIRED`; with a key that doesn't cover it, `403 OUT_OF_SCOPE`. Meshblock
values are never returned — see [Scope, grants & auth](../concepts/scope-and-auth.md#meshblocks-are-roll-up-only).

## Errors

Checked in this order — the first failure wins:

0. `422` — the Area is named both ways, or neither way ([malformed
   request](../concepts/errors.md#malformed-requests))
1. `AREA_NOT_FOUND` (404) — no such AUO Area ID, or `community_code` isn't in the mapping for
   `boundary_type` (and `edition`)
2. `INDICATOR_NOT_FOUND` (404) — indicator key doesn't exist
3. `401`/`403` — you can't see this indicator at any area at all
4. `401`/`403` — you can see the indicator elsewhere, but not at this area
5. `VARIANT_NOT_SUPPORTED` (422) — variant isn't `r`/`p`/`pl`/`pc`

`401` is `AUTHENTICATION_REQUIRED` (no credential sent); `403` is `OUT_OF_SCOPE`.

See [Errors & warnings](../concepts/errors.md) for the shared envelope shape.
