# Indicator metadata

`metadata` — an indicator's definition, provenance and methodology, so a number is never
displayed unexplained.

## One indicator

```
GET /v1/indicators/{key}
```

For example `GET /v1/indicators/walkability_index`:

```json
{
  "key": "walkability_index",
  "domain": "Walkability",
  "description": "Walkability for Transport Index",
  "units": "",
  "polarity": "ASC",
  "source": "auo",
  "decimal_places": 1,
  "percentage_scale": false,
  "variants": ["r", "p", "pl", "pc"],
  "aggregation_method": "weighted_mean_person",
  "custom_region_supported": true,
  "metadata_html": "<p>Walkability is ... <a href='https://...' target='_blank'>...</a></p>",
  "partner_restricted": true,
  "availability": { "lga": [2018, 2021, 2024], "ssc": [2018, 2021, 2024], "sa1": [2018, 2021, 2024] }
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
- `metadata_html` is the indicator's methodology note from the source catalogue, as stored, or
  `null`. It is **HTML** (paragraphs and links, some with `target='_blank'`) and is returned
  unchanged. If you render it, sanitise it first with an allow-list (for example DOMPurify,
  allowing `p`, `a[href]` and little else) and add `rel="noopener noreferrer"` to links that open a
  new tab. Never insert it into a page unsanitised.
- `partner_restricted` is `true` for a partner-restricted indicator: one outside the public
  baseline, visible only where your access covers it. See
  [Scope, grants & auth](../concepts/scope-and-auth.md).
- `availability` lists, per boundary type, the years in which the indicator has at least one
  stored value: `{ "<boundary type>": [years, ascending] }`, coarsest boundary type first, `{}` if
  it has none. It describes the data, not your access: use [My access](my-access.md) to know
  which of these you can see. Meshblock is never listed (meshblock values are roll-up-only), and
  neither are `state` or `study_region`, which have no stored values (a city's average comes from
  [City average](city-average.md)). It is recomputed at most every 10 minutes, so it can lag a
  data load by that long.

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
