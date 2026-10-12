# Errors & warnings

## The error envelope

Every rejected request returns the same shape, regardless of endpoint:

```json
{
  "error": {
    "code": "AREA_NOT_FOUND",
    "message": "No area with AUO Area ID 'lga-999999' exists.",
    "details": { "area": "lga-999999" }
  }
}
```

`details` is always present (an empty object if there's nothing extra to add) — you can
always do `error.details.foo` without a null-check.

## Malformed requests

A request that is structurally wrong is rejected by the framework before any lookup, with
`422` and a different body — a `detail` list, not the envelope above:

```json
{ "detail": [ { "type": "value_error", "loc": ["query", "area"],
                "msg": "Give 'area' (an AUO Area ID), or both 'community_code' and 'boundary_type'.",
                "input": null } ] }
```

This is what you get for a missing required parameter or a wrongly typed one, and for an
[Area reference](areas.md#naming-an-area-in-a-request) that names the Area both ways (`area`
together with `community_code`/`boundary_type`/`edition`), neither way, or `edition` without
`community_code`. `loc` says which field. Code that handles errors should treat any `422` whose
body has `detail` instead of `error` as a request-building bug, not a data condition.

## Whole-request errors

Every code below rejects the request outright with the listed HTTP status. This table is
the single source of truth — individual endpoint pages only call out which of these codes
*that endpoint* can actually return.

| Code | Status | Meaning |
|---|---|---|
| `AREA_NOT_FOUND` | 404 | The named Area doesn't exist: an unknown AUO Area ID, or a `community_code` that isn't in the given `boundary_type` (and `edition`). On a neighbours request it also means the Area has no boundary in that year |
| `INDICATOR_NOT_FOUND` | 404 | Named indicator key doesn't exist |
| `VARIANT_NOT_SUPPORTED` | 422 | Variant isn't one of `r`, `p`, `pl`, `pc` |
| `SCOPE_NOT_FOUND` | 404 | Named ranking/distribution/correlation scope doesn't exist: `scope_code` isn't the AUO Area ID of an Area of the kind `scope` names (an LGA's ID for `lga`, and so on) |
| `INSUFFICIENT_DATA` | 422 | Not enough data to compute the result: fewer than 2 areas for a distribution/correlation, no contributing member for a roll-up, or a roll-up over too few meshblocks (see [Rollup → minimum size](../reference/rollup.md#minimum-size)) |
| `DUPLICATE_INDICATOR` | 422 | Correlation requested between an indicator and itself |
| `CUSTOM_REGION_NOT_SUPPORTED` | 422 | Indicator's `aggregation_method` is `unsupported` — no meshblock source to roll up |
| `CUSTOM_REGION_EMPTY` | 422 | Submitted polygon fully contains zero meshblocks |
| `CUSTOM_REGION_INVALID_GEOMETRY` | 400 | Polygon malformed, self-intersecting, or over 5,000 vertices |
| `CUSTOM_REGION_VARIANT_NOT_SUPPORTED` | 422 | Requested a variant other than `r` against a custom region |
| `AREA_GROUP_NOT_FOUND` | 404 | Named area_group id doesn't exist |
| `INVALID_GEOMETRY` | 400 | Bad GeoJSON, out-of-range lat/lon, or self-intersecting polygon (spatial endpoints) |
| `UNKNOWN_BOUNDARY_TYPE_OR_YEAR` | 400 | That boundary type/year combination doesn't exist in the system |
| `NO_CONTAINING_AREA` | 404 | No area of the requested boundary type/year contains the given point |
| `DOMAIN_NOT_FOUND` | 404 | Named domain filter doesn't exist |
| `QUERY_INVALID` | 400 | Search query empty or over 200 characters (indicator and area search) |
| `AUTHENTICATION_REQUIRED` | 401 | No bearer token, and the request needs one |
| `INVALID_API_KEY` | 401 | Bearer token (an API key) doesn't match a valid, non-revoked key |
| `INVALID_TOKEN` | 401 | Bearer token is an AUO sign-in token that is malformed, wrongly signed or expired; `details.reason` is `invalid` or `expired` |
| `OUT_OF_SCOPE` | 403 | A credential was presented, but your effective scope doesn't cover what the request names |
| `RESOURCE_NOT_FOUND` | 404 | Named computed-resource id (comparison, correlation, region) doesn't exist |
| `INVALID_LIMIT` | 422 | `limit` outside 1–1000 |
| `INSUFFICIENT_AREAS` | 422 | Fewer than 2 Areas in an area comparison |
| `INSUFFICIENT_INDICATORS` | 422 | Fewer than 2 `indicator_keys` in an indicator comparison |
| `UNSUPPORTED_VERSION` | 404 | Version prefix never released |
| `VERSION_WITHDRAWN` | 410 | Version prefix withdrawn |

## Access errors

`AUTHENTICATION_REQUIRED` (401) and `OUT_OF_SCOPE` (403) are the same refusal, split by
whether you sent a credential. Both carry `details.resource_type` (`indicator`, `area`, or a
computed-resource name) and `details.identifier`. They apply to the *whole request*, and an
access error is only ever reported for something that exists — a nonexistent area or
indicator is always a `404` first. Which requests are refused rather than filtered is
explained in [Scope, grants & auth](scope-and-auth.md#how-requests-are-checked).

`INSUFFICIENT_DATA` on a roll-up over meshblocks carries `details.meshblocks` (how many the
region or group contains) and `details.minimum_meshblocks`.

## Item-level notes

One code never rejects the request — it appears inside an otherwise-`200` response, describing
a gap in one item. It is specific to [Trend](../reference/trend.md):

| Code | Meaning |
|---|---|
| `AREA_NOT_IN_YEAR` | The Area has no feature (no boundary, no data) in that year, so the trend has no point for it |

It is reported in the trend's `unavailable_years` list, not as a `null` point:

```json
{
  "unavailable_years": [
    { "year": 2018,
      "reason": { "code": "AREA_NOT_IN_YEAR",
                  "message": "This Area has no feature in 2018, so it has no point for that Year.",
                  "details": { "area": "ssc-004711", "year": 2018 } } }
  ]
}
```

The old `BOUNDARY_CONCORDANCE_EXCEPTION` and `CONCORDANCE_NOT_SUPPORTED` warnings no longer
exist: whether a 2018 value belongs to the same place is now decided by the Area's identity
(see [Areas across years](areas.md#areas-across-years)), not by a per-request mapping.

A response never mixes a top-level `error` with an item-level `reason` — they're mutually
exclusive by construction.

## Validation ordering

When a request has multiple things wrong with it at once, most endpoints check in a fixed
order — area/scope first, then indicator, then variant — and report only the *first*
failure. If you get `INDICATOR_NOT_FOUND`, don't assume the area and variant you sent were
valid; you just haven't found out yet. A [malformed request](#malformed-requests) is reported
before any of these.

Access checks slot in after the existence check of the thing they concern. For a single-value
lookup the order is: area exists → indicator exists → you can see the indicator somewhere →
you can see this indicator at this area → variant is valid. A format check that needs no
database (`limit`, search `q`, coordinates, polygon shape) runs before all of them.
