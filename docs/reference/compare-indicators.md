# Compare indicators

`compare-ind` — 2–20 indicators, side by side, for one area/year/variant. A **computed
resource**, same lifecycle as [Compare areas](compare-areas.md).

## Create

```
POST /v1/comparisons/indicators
```

```json
{
  "indicator_keys": ["walkability", "public_transport_access"],
  "community_code": "20660",
  "boundary_type": "lga",
  "year": 2021,
  "variant": "r"
}
```

### Response — 201

```json
{
  "id": 17,
  "url": "/v1/comparisons/indicators/17",
  "result": {
    "community_code": "20660",
    "year": 2021,
    "variant": "r",
    "indicators": [
      {
        "key": "walkability", "value": 9.1, "domain": "Transport",
        "description": "...", "units": "index score", "polarity": "positive",
        "source": "AUO", "decimal_places": 1, "percentage_scale": false,
        "variants": ["r", "p", "pl", "pc"], "aggregation_method": "weighted_mean_person",
        "custom_region_supported": true
      },
      {
        "key": "public_transport_access", "value": 6.4, "domain": "Transport",
        "description": "...", "units": "% of dwellings", "polarity": "positive",
        "source": "AUO", "decimal_places": 1, "percentage_scale": true,
        "variants": ["r", "p"], "aggregation_method": "weighted_mean_person",
        "custom_region_supported": true
      }
    ]
  }
}
```

Each item carries the same full metadata block as
[indicator metadata](metadata.md) — this is the one endpoint where you get a value and its
methodology/units/polarity together in a single row, useful for rendering a comparison table
without a second round trip.

### Errors (create)

1. `AREA_NOT_FOUND` (404) — `community_code` doesn't resolve for `boundary_type`
2. `INSUFFICIENT_INDICATORS` (422) — fewer than 2 `indicator_keys` — checked before resolving any
3. `INDICATOR_NOT_FOUND` (404) — names every key that failed to resolve
4. `VARIANT_NOT_SUPPORTED` (422) — names the specific affected indicator key

No `OUT_OF_SCOPE` check at creation; enforced on `GET` instead, same as area comparisons.

## Retrieve

```
GET /v1/comparisons/indicators/{comparison_id}
```

Live recompute, scope-intersected with the caller. Supports `Accept: text/csv`.

### Errors (retrieve)

1. `RESOURCE_NOT_FOUND` (404)
2. `OUT_OF_SCOPE` (403)
