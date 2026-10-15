# Area profile

`profile` — every indicator you can see at one area, with every variant, for one year, in a
single call.

```
GET /v1/areas/profile
```

## Parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `area` | query, string | one of `area` / `community_code` + `boundary_type` | The [AUO Area ID](../concepts/areas.md) |
| `community_code` | query, string | with `boundary_type`, instead of `area` | An ABS code, resolved to an Area |
| `boundary_type` | query, string | with `community_code` | |
| `edition` | query, string | no | With `community_code`: the ABS edition to look the code up in |
| `year` | query, integer | yes | |
| `domain` | query, string | no | Restrict to one domain |

The Area is no longer part of the URL path: use `area` (or `community_code` and
`boundary_type`). See [Naming an Area in a
request](../concepts/areas.md#naming-an-area-in-a-request).

## Response — 200

`GET /v1/areas/profile?area=lga-000084&year=2021&domain=Transport`

Trimmed here to the first two of the eleven Transport indicators the call returns; the values are as returned. A `null` is a variant with no recorded value (Yarra has no `pl` percentile).

```json
{
  "area": {
    "area_id": "lga-000084",
    "name": "Yarra",
    "boundary_type": "lga",
    "source_codes": [{ "edition": "LGA 2021", "code": "27350" }, { "edition": "LGA 2016", "code": "27350" }]
  },
  "resolved_from": null,
  "year": 2021,
  "indicators": [
    {
      "key": "transport_closest_bus_with_15_min_service",
      "domain": "Transport",
      "values": { "r": 717.8, "p": 64.0, "pl": null, "pc": 74.0 }
    },
    {
      "key": "transport_closest_bus_with_30_min_service",
      "domain": "Transport",
      "values": { "r": 629.3, "p": 41.0, "pl": null, "pc": 44.0 }
    }
  ]
}
```

`area` and `resolved_from` are as in [Lookup](lookup.md#response-200).

Indicators are ordered by `key`. Only indicators your effective scope covers **at this exact
area** are listed — a public area shows the non-restricted indicators plus any restricted
ones you hold a grant for there. Within the list, this endpoint filters rather than rejects
(see [Scope, grants & auth](../concepts/scope-and-auth.md)). A variant with no recorded
value appears as `null` rather than being omitted — every listed indicator always carries all
four variant keys inside `values`.

## Auth

Public for state, study-region and LGA areas (listing the non-restricted indicators).
For a suburb or SA1, you must have access to at least one indicator at that area; if you have
none, the whole request is refused — `401 AUTHENTICATION_REQUIRED` with no credential,
`403 OUT_OF_SCOPE` with one — rather than returning an empty list. Meshblocks are always
refused.

## Errors

0. `422` — the Area is named both ways, or neither way ([malformed
   request](../concepts/errors.md#malformed-requests))
1. `AREA_NOT_FOUND` (404) — no such AUO Area ID, or `community_code` not found for
   `boundary_type` (and `edition`)
2. `401`/`403` — no indicator is visible to you at this area
3. `DOMAIN_NOT_FOUND` (404) — `domain` filter doesn't exist in the catalogue
