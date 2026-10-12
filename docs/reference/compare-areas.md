# Compare areas

`compare-areas` — the same indicator, side by side, across two or more areas. This is a **computed
resource**: `POST` creates a persistent, shareable result; `GET` retrieves it (live
recomputed against the *caller's own* scope — see
[Scope, grants & auth → computed resources](../concepts/scope-and-auth.md#computed-resources-and-scope)).

## Create

```
POST /v1/comparisons/areas
```

Name the areas by [AUO Area ID](../concepts/areas.md):

```json
{
  "areas": ["lga-000123", "lga-000044", "lga-000310"],
  "indicator": "walkability",
  "year": 2021,
  "variant": "r"
}
```

or by ABS code, in which case `boundary_types` is positional against `community_codes` (areas
may be of different boundary types within the same request) and `edition` optionally fixes the
ABS edition the codes are looked up in:

```json
{
  "community_codes": ["27350", "21310"],
  "boundary_types": ["lga", "lga"],
  "edition": "LGA 2021",
  "indicator": "walkability",
  "year": 2021
}
```

Give `areas` **or** `community_codes` with `boundary_types`, never both. `variant` defaults to
`r`. Each code is resolved to an Area exactly as in a [lookup](lookup.md): the newest ABS
edition containing it unless `edition` says otherwise.

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
      { "area_id": "lga-000123", "name": "Yarra", "boundary_type": "lga",
        "source_codes": [{ "edition": "LGA 2021", "code": "27350" }], "value": 9.1 },
      { "area_id": "lga-000044", "name": "Melbourne", "boundary_type": "lga",
        "source_codes": [{ "edition": "LGA 2021", "code": "24600" }], "value": 8.7 },
      { "area_id": "lga-000310", "name": "Port Phillip", "boundary_type": "lga",
        "source_codes": [{ "edition": "LGA 2021", "code": "25900" }], "value": 7.9 }
    ]
  }
}
```

Each entry is the Area (as everywhere else — `area_id`, `name`, `boundary_type`,
`source_codes`) plus its `value`. The comparison **stores the Areas it resolved**, not the
codes you sent, so it keeps comparing the same places whichever ABS edition a code belonged to.

### Errors (create)

0. `422` — Areas named both ways, `boundary_types` not the same length as `community_codes`, or
   another [malformed request](../concepts/errors.md#malformed-requests)
1. `INSUFFICIENT_AREAS` (422) — fewer than 2 Areas named — checked before resolving any of them
2. `AREA_NOT_FOUND` (404) — names **every** reference that failed to resolve (in
   `details.areas` for AUO Area IDs, `details.community_codes` for codes), not just the first
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
`Accept: text/csv` for a CSV response instead of JSON; in the CSV, the `source_codes`
column is a JSON-encoded list in a single cell.

### Errors (retrieve)

1. `RESOURCE_NOT_FOUND` (404) — no comparison with that id
2. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — none of the stored areas (or not the
   indicator itself) are visible to you
