# AUO Data API

The AUO Data API is a REST service in front of the Australian Urban Observatory's
Postgres/PostGIS database of urban liveability indicators: 206 indicators across 21 domains,
recorded at LGA, suburb (SSC) and SA1 granularity for the 2018, 2021 and 2024 releases, with
geometry for every boundary and year.

This manual is for developers integrating directly with the API — partner organisations,
researchers, and anyone building on top of AUO data rather than using the [AUO map](https://new.map.auo.org.au/)
itself.

## Where to start

- **New to the API?** Start with [Getting Started](getting-started.md) for the base URL,
  versioning, and authentication.
- **Confused by `r`/`p`/`pl`/`pc`, or why a value is `null`?** Read [Concepts](concepts/variants.md)
  first — the six-variant system and the scope model are the two things that trip up most
  new integrations.
- **Know what you want to build, not which endpoint to call?** [Common Tasks](common-tasks.md)
  maps real questions ("which suburb is most liveable?") to concrete API calls.
- **Know exactly which endpoint you need?** Jump straight to [API Reference](reference/lookup.md).

## Capabilities at a glance

| Capability | What it answers | Endpoint(s) |
|---|---|---|
| Lookup | One indicator, one area, one year | `GET /v1/values` |
| Profile | Every indicator for one area | `GET /v1/areas/{code}/profile` |
| Rank | Top/bottom N areas on one indicator | `GET /v1/rankings` |
| Compare areas | Same indicator across several areas | `POST /v1/comparisons/areas` |
| Compare indicators | Several indicators for one area | `POST /v1/comparisons/indicators` |
| Trend | One indicator over 2018/2021/2024 | `GET /v1/areas/{code}/trend` |
| Distribution | Where an area sits among its peers | `GET /v1/distribution` |
| Correlate | Relationship between two indicators | `POST /v1/correlations` |
| Rollup | Aggregate over a health district or drawn polygon | `GET /v1/rollups/area-group/{id}`, `POST /v1/regions` |
| Spatial | Point-in-polygon, adjacency, containment | `GET /v1/spatial/point`, `GET /v1/areas/{code}/neighbours`, `POST /v1/spatial/contains` |
| Search | Find an indicator from a plain-language description | `GET /v1/search/indicators` |
| Metadata | Definition, units, methodology | `GET /v1/indicators`, `GET /v1/indicators/{key}` |
| Admin | Grants, area groups, coverage, API keys | `GET/POST/DELETE /v1/admin/...` |

Every endpoint is under a version prefix (`/v1` today) — see
[API Versioning](getting-started.md#versioning).
