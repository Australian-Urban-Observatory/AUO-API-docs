# Compare areas

`compare-areas` — the same indicator, side by side, across two or more areas. This is a **computed
resource**: `POST` creates a persistent, shareable result; `GET` retrieves it (live
recomputed against the *caller's own* scope — see
[Scope, grants & auth → computed resources](../concepts/scope-and-auth.md#computed-resources-and-scope)).

## Create

```
POST /v1/comparisons/areas
```

```json
{
  "community_codes": ["20660", "20140", "20310"],
  "boundary_types": ["lga", "lga", "lga"],
  "indicator": "walkability",
  "year": 2021,
  "variant": "r"
}
```

`boundary_types` is positional against `community_codes` — areas may be of different
boundary types within the same request, each evaluated at its own type. `variant` defaults
to `r`.

### Response — 201

```json
{
  "id": 42,
  "url": "/v1/comparisons/areas/42",
  "result": {
    "indicator": "walkability",
    "year": 2021,
    "variant": "r",
    "areas": [
      { "community_code": "20660", "boundary_type": "lga", "value": 9.1 },
      { "community_code": "20140", "boundary_type": "lga", "value": 8.7 },
      { "community_code": "20310", "boundary_type": "lga", "value": 7.9 }
    ]
  }
}
```

### Errors (create)

1. `INSUFFICIENT_AREAS` (422) — fewer than 2 `community_codes` — checked before resolving any of them
2. `AREA_NOT_FOUND` (404) — names **every** code that failed to resolve, not just the first
3. `INDICATOR_NOT_FOUND` (404)
4. `VARIANT_NOT_SUPPORTED` (422)

Access is not checked area by area at creation: you can name any existing areas. But the
`result` in the `201` is computed against **your** scope, so areas you can't see are left out
of it, and if you can see none of them (or not the indicator) the request is refused with
`401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` instead of a `201`.

## Retrieve

```
GET /v1/comparisons/areas/{comparison_id}
```

Recomputes live from the stored inputs, restricted to the intersection of those inputs and
**your** effective scope — an area is included only if you can see this indicator at it.
Areas you can't see are omitted from `areas` (not returned as `null`). The retrieved body is
`{indicator, year, variant, areas}`, without the `id`/`url`/`result` wrapper. Supports
`Accept: text/csv` for a CSV response instead of JSON.

### Errors (retrieve)

1. `RESOURCE_NOT_FOUND` (404) — no comparison with that id
2. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — none of the stored areas (or not the
   indicator itself) are visible to you
