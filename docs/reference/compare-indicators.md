# Compare indicators

`compare-indicators` — two or more indicators, side by side, for one area/year/variant. A **computed
resource**, same lifecycle as [Compare areas](compare-areas.md).

## Create

```
POST /v1/comparisons/indicators
```

Name the area by [AUO Area ID](../concepts/areas.md):

```json
{
  "indicator_keys": ["walkability", "public_transport_access"],
  "area": "lga-000123",
  "year": 2021,
  "variant": "r"
}
```

or by ABS code (`community_code` and `boundary_type`, and optionally `edition`):

```json
{
  "indicator_keys": ["walkability", "public_transport_access"],
  "community_code": "27350",
  "boundary_type": "lga",
  "year": 2021
}
```

Give one form or the other, never both. `variant` defaults to `r`.

### Response — 201

```json
{
  "id": 17,
  "url": "/v1/comparisons/indicators/17",
  "result": {
    "area": {
      "area_id": "lga-000123",
      "name": "Yarra",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "27350" }]
    },
    "year": 2021,
    "variant": "r",
    "indicators": [
      {
        "key": "walkability", "value": 9.1, "domain": "Transport",
        "description": "...", "units": "index score", "polarity": "ASC",
        "source": "AUO", "decimal_places": 1, "percentage_scale": false,
        "variants": ["r", "p", "pl", "pc"], "aggregation_method": "weighted_mean_person",
        "custom_region_supported": true
      },
      {
        "key": "public_transport_access", "value": 6.4, "domain": "Transport",
        "description": "...", "units": "% of dwellings", "polarity": "ASC",
        "source": "AUO", "decimal_places": 1, "percentage_scale": true,
        "variants": ["r", "p", "pl", "pc"], "aggregation_method": "weighted_mean_person",
        "custom_region_supported": true
      }
    ]
  }
}
```

`area` describes the Area the comparison is about (the same shape as in [Lookup](lookup.md)),
in place of the former top-level `community_code`. The comparison stores the Area it resolved,
so it keeps comparing the same place.

Each item carries the same full metadata block as
[indicator metadata](metadata.md) — this is the one endpoint where you get a value and its
methodology/units/polarity together in a single row, useful for rendering a comparison table
without a second round trip.

### Errors (create)

0. `422` — the Area is named both ways, or neither way ([malformed
   request](../concepts/errors.md#malformed-requests))
1. `AREA_NOT_FOUND` (404) — no such AUO Area ID, or `community_code` not found for
   `boundary_type` (and `edition`)
2. `INSUFFICIENT_INDICATORS` (422) — fewer than 2 `indicator_keys` — checked before resolving any
3. `INDICATOR_NOT_FOUND` (404) — names every key that failed to resolve
4. `VARIANT_NOT_SUPPORTED` (422) — the variant isn't `r`/`p`/`pl`/`pc`

Access is applied the same way as [area comparisons](compare-areas.md#create): the inline
`result` only contains indicators you can see at this area, and if there are none you get
`401`/`403` instead of a `201`.

## Retrieve

```
GET /v1/comparisons/indicators/{comparison_id}
```

Live recompute, scope-intersected with the caller: an indicator is included only if you can
see it at the stored area, and the rest are omitted. The body is `{area, year, variant,
indicators}`. Supports `Accept: text/csv`.

### Errors (retrieve)

1. `RESOURCE_NOT_FOUND` (404)
2. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — you can see none of the stored
   indicators at that area
