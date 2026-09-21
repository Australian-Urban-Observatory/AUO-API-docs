# Lookup a value

`lookup` — one indicator, one area, one year, one variant.

```
GET /v1/values
```

## Query parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `community_code` | string | yes | |
| `boundary_type` | string | yes | e.g. `lga`, `ssc`, `sa1` |
| `indicator` | string | yes | Indicator key |
| `year` | integer | yes | |
| `variant` | string | no | Default `r`. See [Variants](../concepts/variants.md) |

## Response — 200

```json
{
  "community_code": "20660",
  "boundary_type": "lga",
  "indicator": "walkability",
  "year": 2021,
  "variant": "r",
  "value": 4.2,
  "polarity": "positive",
  "units": "index score"
}
```

`value` is `null` if no value is recorded for this exact combination — see
[Variants → null values](../concepts/variants.md#null-values). `polarity` and `units` come
from the indicator's own metadata and are always included.

## Auth

Public unless the named indicator is partner-restricted or the named area is
access-grant-restricted — in which case a valid, in-scope key is required, or the request
gets `403 OUT_OF_SCOPE` (or `401` with no/invalid key).

## Errors

Checked in this order — the first failure wins:

1. `AREA_NOT_FOUND` (404) — `community_code` doesn't exist for `boundary_type`
2. `403 OUT_OF_SCOPE` — area exists but isn't in your effective scope
3. `INDICATOR_NOT_FOUND` (404) — indicator key doesn't exist
4. `403 OUT_OF_SCOPE` — indicator exists but isn't in your effective scope
5. `VARIANT_NOT_SUPPORTED` (422) — variant not defined for this indicator

See [Errors & warnings](../concepts/errors.md) for the shared envelope shape.
