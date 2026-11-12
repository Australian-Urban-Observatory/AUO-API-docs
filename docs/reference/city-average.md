# City average

`city-average` — an indicator's average over an area's **city** (its study region, e.g.
Melbourne), for showing an area's value beside its city's.

```
GET /v1/values/city
```

## Query parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `area` | string | yes | The [AUO Area ID](../concepts/areas.md) of any area: a study region, or an LGA, suburb or SA1 in one |
| `indicator` | string | yes | Indicator key |
| `year` | integer | yes | |

## Response — 200

`GET /v1/values/city?area=lga-000049&indicator=urban_liveability_index&year=2021`

```json
{
  "indicator": "urban_liveability_index",
  "year": 2021,
  "variant": "r",
  "area_id": "lga-000049",
  "study_region": { "area_id": "study_region-000014", "name": "Melbourne" },
  "value": 98.821
}
```

- The city is the study region the area is **assigned to** in that year (a study region is its
  own city). `study_region` is `null` if the area has none that year, and then `value` is `null`
  too.
- **Every 2018 city average is `null`.** 2018 areas have no assigned parents, so no 2018
  meshblock belongs to a city. A 2018 request still succeeds (200) with `"value": null`, and with
  `"study_region": null` unless the area you name is itself a study region.
- `value` is the population-weighted mean of the raw value (`r`) over the city's meshblocks,
  rounded to 3 decimal places. Only the raw variant has a city average: an average of
  percentiles is not a percentile. `null` means no data for that city and year.
- Only indicators with `"custom_region_supported": true` in the
  [catalogue](metadata.md) have a city average (they are the ones with meshblock-level data).
- Results may be cached for up to 10 minutes, so a city average can lag a data update by that
  long. Access is still checked on every request.

## Auth

A city average counts as the study region's own value. You get it when you can see the
indicator at that study region: anyone for a public indicator (study regions are in the public
baseline), and only with access that covers that study region for a partner-restricted one.
Access to the indicator at suburbs or SA1s inside the city is not enough. The `area` you name
must also be one you can see: an LGA or study region for anyone, a suburb or SA1 only with access
there, never a meshblock. Meshblock values themselves are never returned. See [Scope, grants & auth](../concepts/scope-and-auth.md).

## Errors

- `AREA_NOT_FOUND` (404) — `area` isn't an AUO Area ID.
- `INDICATOR_NOT_FOUND` (404) — the indicator key doesn't exist.
- `CUSTOM_REGION_NOT_SUPPORTED` (422) — the indicator has no meshblock-level data, so it has no
  city average.
- `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — you can't see the named area, you can't
  see the indicator anywhere, or you can't see it at the area's study region.
- `REQUEST_INVALID` (422) — a required parameter is missing or mistyped.

Also the usual [version-prefix errors](../getting-started.md#versioning).
