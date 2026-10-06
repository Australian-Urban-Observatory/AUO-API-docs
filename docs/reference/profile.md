# Area profile

`profile` — every indicator you can see at one area, with every variant, for one year, in a
single call.

```
GET /v1/areas/{community_code}/profile
```

## Parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `community_code` | path | yes | |
| `year` | query, integer | yes | |
| `boundary_type` | query, string | yes | |
| `domain` | query, string | no | Restrict to one domain |

## Response — 200

```json
{
  "community_code": "20660",
  "year": 2021,
  "indicators": [
    {
      "key": "walkability",
      "domain": "Transport",
      "values": { "r": 4.2, "p": 71.0, "pl": 55.0, "pc": 62.0 }
    },
    {
      "key": "public_transport_access",
      "domain": "Transport",
      "values": { "r": null, "p": null, "pl": null, "pc": null }
    }
  ]
}
```

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

1. `AREA_NOT_FOUND` (404) — `community_code` doesn't exist for `boundary_type`
2. `401`/`403` — no indicator is visible to you at this area
3. `DOMAIN_NOT_FOUND` (404) — `domain` filter doesn't exist in the catalogue
