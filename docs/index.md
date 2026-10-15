# AUO Data API

The AUO Data API is a REST service in front of the Australian Urban Observatory's
Postgres/PostGIS database of urban liveability indicators: the full catalogue is 275
indicators across 20 domains,
recorded at LGA, suburb (SSC) and SA1 granularity for the 2018, 2021 and 2024 releases, with
geometry for every boundary and year. Non-restricted indicators are open at state, study-region
and LGA level with no key; suburb and SA1 data, and partner-restricted indicators, need an
API key.

The catalogue you see depends on your access: `GET /v1/indicators` omits indicators you
can't see at any area, so an unauthenticated request returns fewer (218 indicators across 15
domains when last checked) than the full 275. See
[Metadata](reference/metadata.md#full-catalogue) and the [scope model](concepts/scope-and-auth.md).

This manual is for developers integrating directly with the API — partner organisations,
researchers, and anyone building on top of AUO data rather than using the [AUO map](https://new.map.auo.org.au/)
itself.

## Where to start

- **New to the API?** Start with [Getting Started](getting-started.md) for the base URL,
  versioning, and authentication.
- **Confused by `r`/`p`/`pl`/`pc`, or why a value is `null`, or why you got a `401`/`403`?**
  Read [Concepts](concepts/variants.md) first — the four-variant system and the
  [scope model](concepts/scope-and-auth.md) are the two things that trip up most new
  integrations.
- **Not sure how to name a place?** [Areas & AUO Area IDs](concepts/areas.md) explains the
  permanent ID every endpoint uses, how it relates to ABS codes, and how areas persist across
  releases.
- **Know what you want to build, not which endpoint to call?** [Common Tasks](common-tasks.md)
  maps real questions ("which suburb is most liveable?") to concrete API calls.
- **Know exactly which endpoint you need?** Jump straight to [API Reference](reference/lookup.md).

## Capabilities at a glance

| Capability | What it answers | Endpoint(s) |
|---|---|---|
| Lookup | One indicator, one area, one year | `GET /v1/values` |
| Profile | Every indicator for one area | `GET /v1/areas/profile` |
| Rank | Top/bottom N areas on one indicator | `GET /v1/rankings` |
| Bulk values | Every area's value for one indicator, in one response | `GET /v1/values/bulk` |
| Compare areas | Same indicator across several areas | `POST /v1/comparisons/areas` |
| Compare indicators | Several indicators for one area | `POST /v1/comparisons/indicators` |
| Trend | One indicator for one area across the releases it exists in | `GET /v1/areas/trend` |
| Distribution | Where an area sits among its peers | `GET /v1/distribution` |
| Correlate | Relationship between two indicators | `POST /v1/correlations` |
| Rollup | Aggregate over a health district or drawn polygon | `GET /v1/rollups/area-group/{id}`, `POST /v1/regions` |
| Spatial | Point-in-polygon, adjacency, containment | `GET /v1/spatial/point`, `GET /v1/areas/neighbours`, `POST /v1/spatial/contains` |
| Search indicators | Find an indicator from a plain-language description | `GET /v1/search/indicators` |
| Search areas | Find an area's AUO Area ID from a place name | `GET /v1/search/areas` |
| Metadata | Definition, units, methodology | `GET /v1/indicators`, `GET /v1/indicators/{key}` |

Every endpoint is under a version prefix (`/v1` today) — see
[API Versioning](getting-started.md#versioning).

!!! info "Areas are identified by AUO Area ID"
    Places are named by a permanent **AUO Area ID** (`area=lga-000084`), not by an ABS
    code, because an ABS code can mean a different place in another ABS edition. ABS codes are
    still accepted as a lookup. See [Areas & AUO Area IDs](concepts/areas.md).
