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

Public only for a non-restricted indicator at state, study-region or LGA level. Suburb and
SA1 values, and partner-restricted indicators anywhere, need a key whose access covers that
exact indicator at that exact area. Without a key the request gets `401
AUTHENTICATION_REQUIRED`; with a key that doesn't cover it, `403 OUT_OF_SCOPE`. Meshblock
values are never returned — see [Scope, grants & auth](../concepts/scope-and-auth.md#meshblocks-are-roll-up-only).

## Errors

Checked in this order — the first failure wins:

1. `AREA_NOT_FOUND` (404) — `community_code` doesn't exist for `boundary_type`
2. `INDICATOR_NOT_FOUND` (404) — indicator key doesn't exist
3. `401`/`403` — you can't see this indicator at any area at all
4. `401`/`403` — you can see the indicator elsewhere, but not at this area
5. `VARIANT_NOT_SUPPORTED` (422) — variant isn't `r`/`p`/`pl`/`pc`, or `pl`/`pc` isn't
   available for this area

`401` is `AUTHENTICATION_REQUIRED` (no credential sent); `403` is `OUT_OF_SCOPE`.

See [Errors & warnings](../concepts/errors.md) for the shared envelope shape.
