# Errors & warnings

## The error envelope

Every rejected request returns the same shape, regardless of endpoint:

```json
{
  "error": {
    "code": "AREA_NOT_FOUND",
    "message": "No area with community_code '99999' exists for boundary type 'lga'.",
    "details": { "community_code": "99999", "boundary_type": "lga" }
  }
}
```

`details` is always present (an empty object if there's nothing extra to add) — you can
always do `error.details.foo` without a null-check.

## Whole-request errors

Every code below rejects the request outright with the listed HTTP status. This table is
the single source of truth — individual endpoint pages only call out which of these codes
*that endpoint* can actually return.

| Code | Status | Meaning |
|---|---|---|
| `AREA_NOT_FOUND` | 404 | Named `community_code` doesn't exist for that boundary type/year |
| `INDICATOR_NOT_FOUND` | 404 | Named indicator key doesn't exist |
| `VARIANT_NOT_SUPPORTED` | 422 | Variant not defined for this indicator, or not valid for this area (e.g. `pl` with no LGA parent) |
| `SCOPE_NOT_FOUND` | 404 | Named ranking/distribution/correlation scope (state, study region, LGA code) doesn't exist |
| `INSUFFICIENT_DATA` | 422 | Fewer than 2 areas have data — for a distribution/correlation, or after scope filtering removed every contributing area |
| `DUPLICATE_INDICATOR` | 422 | Correlation requested between an indicator and itself |
| `CUSTOM_REGION_NOT_SUPPORTED` | 422 | Indicator's `aggregation_method` is `unsupported` — no meshblock source to roll up |
| `CUSTOM_REGION_EMPTY` | 422 | Submitted polygon fully contains zero meshblocks |
| `CUSTOM_REGION_INVALID_GEOMETRY` | 400 | Polygon malformed, self-intersecting, or over 5,000 vertices |
| `CUSTOM_REGION_VARIANT_NOT_SUPPORTED` | 422 | Requested a variant other than `r` against a custom region |
| `AREA_GROUP_NOT_FOUND` | 404 | Named area_group id doesn't exist |
| `INVALID_GEOMETRY` | 400 | Bad GeoJSON, out-of-range lat/lon, or self-intersecting polygon (spatial endpoints) |
| `UNKNOWN_BOUNDARY_TYPE_OR_YEAR` | 400 (404 for admin coverage) | That boundary type/year combination doesn't exist in the system |
| `NO_CONTAINING_AREA` | 404 | No area of the requested boundary type/year contains the given point |
| `DOMAIN_NOT_FOUND` | 404 | Named domain filter doesn't exist |
| `QUERY_INVALID` | 400 | Search query empty or over 200 characters |
| `AUTHENTICATION_REQUIRED` | 401 | No bearer token, and the request needs one |
| `INVALID_API_KEY` | 401 | Bearer token doesn't match a valid, non-revoked key |
| `OUT_OF_SCOPE` | 403 | Request targets an indicator/area/resource outside your effective scope |
| `RESOURCE_NOT_FOUND` | 404 | Named computed-resource id (comparison, correlation, region) doesn't exist |
| `INVALID_LIMIT` | 422 | `limit` outside 1–1000 |
| `INSUFFICIENT_AREAS` | 422 | Fewer than 2 `community_codes` in an area comparison |
| `INSUFFICIENT_INDICATORS` | 422 | Fewer than 2 `indicator_keys` in an indicator comparison |
| `GRANTEE_NOT_FOUND` | 404 | Named grantee has no grants at all (admin endpoints) |
| `UNSUPPORTED_VERSION` | 404 | Version prefix never released |
| `VERSION_WITHDRAWN` | 410 | Version prefix withdrawn |

## Item-level warnings

Two codes never reject the request — they appear as a `warning` field on one item inside an
otherwise-`200` response, alongside a `null` value for that item. Both are specific to
[Trend](../reference/trend.md)'s 2018 leg:

| Code | Meaning |
|---|---|
| `BOUNDARY_CONCORDANCE_EXCEPTION` | This LGA has no stored concordance mapping to a 2018 boundary |
| `CONCORDANCE_NOT_SUPPORTED` | Cross-year concordance isn't defined at this boundary type at all (SSC, SA1) |

```json
{
  "indicator": "walkability",
  "variant": "r",
  "points": [
    { "year": 2018, "value": null, "warning": { "code": "CONCORDANCE_NOT_SUPPORTED", "message": "...", "details": {} } },
    { "year": 2021, "value": 4.1 },
    { "year": 2024, "value": 4.3 }
  ]
}
```

A response never mixes a top-level `error` with an item-level `warning` — they're mutually
exclusive by construction.

## Validation ordering

When a request has multiple things wrong with it at once, most endpoints check in a fixed
order — area/scope first, then indicator, then variant — and report only the *first*
failure. If you get `INDICATOR_NOT_FOUND`, don't assume the area and variant you sent were
valid; you just haven't found out yet.
