# Indicator metadata

`metadata` — an indicator's definition, provenance and methodology, so a number is never
displayed unexplained.

## One indicator

```
GET /v1/indicators/{key}
```

```json
{
  "key": "walkability",
  "domain": "Transport",
  "description": "Composite score of street connectivity, dwelling density, and daily living access.",
  "units": "index score",
  "polarity": "positive",
  "source": "AUO",
  "decimal_places": 1,
  "percentage_scale": false,
  "variants": ["r", "p", "pl", "pc"],
  "aggregation_method": "weighted_mean_person",
  "custom_region_supported": true
}
```

- `polarity` tells you whether a higher raw value is good (`positive`) or bad (`negative`) —
  needed to interpret `r` correctly; ranking already accounts for this automatically (see
  [Rank](rank.md)).
- `custom_region_supported` is derived directly from `aggregation_method`: `true` if and
  only if it's `weighted_mean_person`. If `false`, this indicator can never be used in a
  [custom-region rollup](rollup.md#custom-region-rollup).
- `variants` lists exactly which of `r`/`p`/`pl`/`pc` this indicator defines — request
  anything outside this list and every other endpoint returns `VARIANT_NOT_SUPPORTED`.

**Errors:** `INDICATOR_NOT_FOUND` (404); `OUT_OF_SCOPE` (403) — the key exists but names a
partner-restricted indicator outside your effective scope.

## Full catalogue

```
GET /v1/indicators
```

```json
{ "indicators": [ /* one object per indicator, same shape as above */ ] }
```

Filtered to your effective scope — no rejection, just omission. An unauthenticated request
never sees partner-restricted indicators here at all.
