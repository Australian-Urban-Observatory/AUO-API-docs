# Area profile

`profile` — every indicator, every variant, for one area and year, in a single call.

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
    { "key": "walkability", "r": 4.2, "p": 71.0, "pl": 55.0, "pc": 62.0 },
    { "key": "public_transport_access", "r": null, "p": null, "pl": null, "pc": null }
  ]
}
```

Only indicators within **your** effective scope are included — this endpoint filters rather
than rejects (see [Scope, grants & auth](../concepts/scope-and-auth.md)). A variant an
indicator doesn't define, or has no recorded value for, appears as `null` rather than being
omitted — every listed indicator always carries all four variant keys.

## Auth

Public, but the indicator list returned is scoped to whatever your key (or lack of one)
authorises. There is no area-level `OUT_OF_SCOPE` rejection on this endpoint — only
indicator-level filtering.

## Errors

1. `AREA_NOT_FOUND` (404) — `community_code` doesn't exist for `boundary_type`/`year`
2. `DOMAIN_NOT_FOUND` (404) — `domain` filter doesn't exist in the catalogue
