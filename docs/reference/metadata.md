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
  "polarity": "ASC",
  "source": "AUO",
  "decimal_places": 1,
  "percentage_scale": false,
  "variants": ["r", "p", "pl", "pc"],
  "aggregation_method": "weighted_mean_person",
  "custom_region_supported": true
}
```

- `polarity` tells you which direction of the raw value is the better one: `ASC` means a
  higher raw value is better, `DESC` means a lower one is. It is returned exactly as stored.
  It is needed to interpret `r` correctly; ranking already accounts for this automatically
  (see [Rank](rank.md)), reading any other or empty value as "higher is better". Percentile
  variants already have polarity built in (100 is always best).
- `custom_region_supported` is derived directly from `aggregation_method`: `true` if and
  only if it's `weighted_mean_person`. If `false`, this indicator can never be used in a
  [custom-region rollup](rollup.md#custom-region-rollup).
- `variants` is always `["r", "p", "pl", "pc"]` — the fixed set every endpoint accepts.
  An indicator having no recorded value for a variant shows up as `null` in the data, not
  as a shorter list here.

**Errors:** `INDICATOR_NOT_FOUND` (404); `401 AUTHENTICATION_REQUIRED` (no credential) /
`403 OUT_OF_SCOPE` (credential presented) — the key exists but you can't see the indicator at
any area, as is the case for a partner-restricted indicator outside your access. The
indicator's existence isn't hidden; its metadata is.

## Full catalogue

```
GET /v1/indicators
```

```json
{ "indicators": [ /* one object per indicator, same shape as above */ ] }
```

Filtered to indicators you can see at one or more areas — no rejection, just omission. An
unauthenticated request never sees partner-restricted indicators here at all.
