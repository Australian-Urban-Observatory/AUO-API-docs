# Rollup

`rollup` — aggregate an indicator up to a scope the underlying data doesn't natively
partition on: a named **area group** (e.g. a Local Health District), or a **custom-drawn
polygon**.

Both mechanisms only work for indicators whose `aggregation_method` is
`weighted_mean_person` (check `custom_region_supported` on
[indicator metadata](metadata.md)). Census/ABS and `housing_*` indicators have no
meshblock-level source and can't be rolled up this way at all.

## Area group rollup

```
GET /v1/rollups/area-group/{area_group_id}?indicator={key}&year={year}&variant={r|p|pl|pc}
```

Aggregates the values of the group's member areas, restricted to members within **your**
effective scope, excluding any member with no recorded value for that indicator/year/variant.

### Response — 200

```json
{
  "area_group_id": 3,
  "indicator": "walkability",
  "year": 2021,
  "variant": "r",
  "value": 5.8,
  "member_count": 11
}
```

`member_count` is the number of members that actually **contributed** a value — not the
group's total membership.

### Errors

1. `AREA_GROUP_NOT_FOUND` (404)
2. `INDICATOR_NOT_FOUND` (404)
3. `VARIANT_NOT_SUPPORTED` (422)
4. `INSUFFICIENT_DATA` (422) — no in-scope member has a value for this indicator/year/variant

## Custom region rollup

A custom region is a persisted resource resolved once, at creation, to the set of meshblocks
fully contained within a submitted polygon (up to 5,000 vertices). Unlike comparisons and
correlations, only the **value** endpoint recomputes on every `GET` — membership itself is
resolved once and cached, not part of the computed-resource scope-intersection lifecycle.

### Create a region

```
POST /v1/regions
```

```json
{ "polygon": { "type": "Polygon", "coordinates": [[...]] }, "label": "Proposed precinct A" }
```

```json
{ "id": 91, "url": "/v1/regions/91", "meshblock_count": 47 }
```

`label` is optional. If the polygon resolves to zero meshblocks, nothing is created —
`meshblock_count` is never `0` for a region you can subsequently fetch.

**Errors:** `CUSTOM_REGION_INVALID_GEOMETRY` (400) — malformed, self-intersecting, or over
5,000 vertices; `CUSTOM_REGION_EMPTY` (422) — valid polygon, zero meshblocks fully contained.

### Get a region's value for an indicator

```
GET /v1/regions/{region_id}/values?indicator={key}&year={year}&variant=r
```

Only `r` is ever supported here — see
[Variants → custom regions only ever support `r`](../concepts/variants.md#custom-regions-only-ever-support-r).
Computes the person-weighted mean of the region's meshblock-level values, live, on every call.

```json
{
  "region_id": 91,
  "indicator": "walkability",
  "year": 2021,
  "variant": "r",
  "value": 6.1,
  "meshblock_count": 44
}
```

`meshblock_count` here is the number of **contributing** meshblocks (survived scope + value +
weight filtering) — it can be lower than the region's total membership reported at creation.

**Errors:**

1. `RESOURCE_NOT_FOUND` (404) — no region with that id
2. `INDICATOR_NOT_FOUND` (404)
3. `CUSTOM_REGION_NOT_SUPPORTED` (422) — indicator's `aggregation_method` is `unsupported`
4. `CUSTOM_REGION_VARIANT_NOT_SUPPORTED` (422) — anything other than `r`
5. `INSUFFICIENT_DATA` (422) — every member meshblock excluded by scope, missing value, or
   missing weight (distinct from `CUSTOM_REGION_EMPTY`, which can only happen at creation —
   a region that exists always has at least one member)
