# Trend

`trend` — one indicator, one area, across all three releases (2018, 2021, 2024).

```
GET /v1/areas/{community_code}/trend
```

## Parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `community_code` | path | yes | |
| `indicator` | query, string | yes | |
| `boundary_type` | query, string | yes | |
| `variant` | query, string | no | Default `r` |

## Response — 200

```json
{
  "indicator": "walkability",
  "variant": "r",
  "points": [
    { "year": 2018, "value": 3.9 },
    { "year": 2021, "value": 4.1 },
    { "year": 2024, "value": 4.3 }
  ]
}
```

A release with no recorded value returns `value: null` for that point rather than omitting
it — `points` always has exactly three entries.

## The 2018 leg

2021 and 2024 always resolve directly against your `community_code`. 2018 is different,
because boundaries changed between releases — see
[Boundary hierarchy → cross-year concordance](../concepts/boundary-hierarchy.md#cross-year-boundary-concordance):

- **LGA, with a stored concordance mapping:** 2018 resolves via that mapping. Normal `value`,
  no warning.
- **LGA, no concordance mapping:** `value: null` for 2018, plus an item-level warning:

    ```json
    { "year": 2018, "value": null, "warning": { "code": "BOUNDARY_CONCORDANCE_EXCEPTION", "message": "...", "details": {} } }
    ```

- **SSC or SA1 (any):** concordance isn't defined at all below LGA. 2018 always comes back
  as `value: null` with:

    ```json
    { "year": 2018, "value": null, "warning": { "code": "CONCORDANCE_NOT_SUPPORTED", "message": "...", "details": {} } }
    ```

Either way, the overall response is still `200` — a 2018 concordance gap never fails the
whole request, and 2021/2024 are unaffected.

## Auth

Rejecting, not filtering — same as [Lookup](lookup.md): naming an out-of-scope area or
indicator directly gets `403 OUT_OF_SCOPE`.

## Errors

1. `AREA_NOT_FOUND` (404)
2. `OUT_OF_SCOPE` (403) — area
3. `INDICATOR_NOT_FOUND` (404)
4. `OUT_OF_SCOPE` (403) — indicator
5. `VARIANT_NOT_SUPPORTED` (422)
