# Rollup

`rollup` — aggregate an indicator up to a scope the underlying data doesn't natively
partition on: a named **area group** (e.g. a Local Health District), or a **custom-drawn
polygon**.

The two mechanisms work differently:

| | Area group | Custom region |
|---|---|---|
| Built from | The group's member areas (any boundary type) | Meshblocks fully inside your polygon |
| Aggregate | Unweighted mean of the members' values | Person-weighted mean of meshblock values |
| Indicators | Any | Only where `custom_region_supported` is `true` ([metadata](metadata.md)) |
| Variants | `r`, `p`, `pl`, `pc` | `r` only |

Census/ABS and `housing_*` indicators have no meshblock-level source, so they can't be used
in a custom region. They can still appear in an area-group rollup.

## Access

Roll-ups are the one place meshblock data is used, and it is used only as input to an
aggregate — raw meshblock values are never returned ([why](../concepts/scope-and-auth.md#meshblocks-are-roll-up-only)).
A roll-up draws on meshblocks only if your access covers them for that indicator; public
access does not. So:

- A **custom region** needs access that covers meshblocks. With no credential you get `401
  AUTHENTICATION_REQUIRED`; with a credential that doesn't reach meshblocks, `403
  OUT_OF_SCOPE`.
- An **area group** made of LGAs or suburbs rolls up over the members you can see. Members
  you can't see are left out; if that leaves none, the request is refused with `401`/`403`
  rather than `INSUFFICIENT_DATA`.

## Minimum size

A roll-up over very few meshblocks is close to a raw meshblock value, so it is refused unless
enough of them actually contribute. The default minimum is **5**. It is counted *after*
dropping meshblocks that are out of your scope or have no value or no population weight —
so a region can contain more than 5 meshblocks and still fall short.

When the minimum isn't met you get `422 INSUFFICIENT_DATA` with no value, and the error
`details` tell you how far short you are:

```json
{
  "error": {
    "code": "INSUFFICIENT_DATA",
    "message": "...",
    "details": { "region_id": 91, "indicator": "walkability", "year": 2021,
                 "variant": "r", "contributing_meshblocks": 3, "minimum_meshblocks": 5 }
  }
}
```

For an area group the minimum applies to its meshblock members whenever any of them
contributes; adding LGA or suburb members doesn't satisfy it. A group with no meshblock
members isn't affected.

## Area group rollup

```
GET /v1/rollups/area-group/{area_group_id}?indicator={key}&year={year}&variant={r|p|pl|pc}
```

Area groups are predefined by the AUO team; ask for the id of the group you need.

Averages the values of the group's member areas (an unweighted mean, rounded to the
indicator's precision), restricted to members within **your** effective scope and excluding
any member with no recorded value for that indicator/year/variant. Members aren't expanded —
a member that is an LGA contributes the LGA's own value, not its suburbs'.

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
3. `401`/`403` — you can't see the indicator at any area
4. `VARIANT_NOT_SUPPORTED` (422)
5. `401`/`403` — the group has members, but none are visible to you
6. `INSUFFICIENT_DATA` (422) — no visible member has a value for this indicator/year/variant
7. `INSUFFICIENT_DATA` (422) — too few meshblock members contribute ([minimum size](#minimum-size))

## Custom region rollup

A custom region is a persisted resource resolved once, at creation, to the set of meshblocks
fully contained within a submitted polygon (up to 5,000 vertices). Membership is resolved
once and stored; the **value** endpoint recomputes on every `GET`.

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

`label` is optional. Creating a region doesn't need a key and doesn't read any values — the
access check happens when you ask for values. If the polygon resolves to zero meshblocks,
nothing is created, so `meshblock_count` is never `0` for a region you can subsequently
fetch.

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
5. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — you can't see the indicator anywhere,
   or your access doesn't cover any of the region's meshblocks for it
6. `INSUFFICIENT_DATA` (422) — fewer than the [minimum](#minimum-size) meshblocks contribute
7. `INSUFFICIENT_DATA` (422) — every member meshblock was excluded by scope, missing value, or
   missing weight (distinct from `CUSTOM_REGION_EMPTY`, which can only happen at creation —
   a region that exists always has at least one member)
