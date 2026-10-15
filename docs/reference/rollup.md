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

A roll-up over very few meshblocks is close to a raw meshblock value (a region containing one
meshblock would return that meshblock's value exactly), so a roll-up is refused unless its
region holds enough of them. The minimum is not published: the API doesn't tell you the
threshold, and never reports how many meshblocks a region or group contains.

The count is of the meshblocks **in the region**, before anything is filtered out for your
scope, a missing value or a missing population weight. A region that meets the minimum is
never refused for size, even if only a few of its meshblocks end up contributing. A region
or group with **no** meshblocks at all isn't a meshblock roll-up and isn't subject to the minimum.

When the minimum isn't met you get `422 INSUFFICIENT_DATA` with no value. The message is
always "The study area is too small to return a value." and the error `details` only echo
your request:

```json
{
  "error": {
    "code": "INSUFFICIENT_DATA",
    "message": "The study area is too small to return a value.",
    "details": { "region_id": 903, "indicator": "urban_liveability_index", "year": 2021, "variant": "r" }
  }
}
```

For an area group the `details` carry `area_group_id` in place of `region_id`. To get a value,
draw a larger region or use a larger group.

For an area group the minimum applies to the group's **meshblock members** whenever it has any;
adding LGA or suburb members doesn't satisfy it, because their values are public and could
be subtracted from the mean to recover the meshblocks'. A group with no meshblock members isn't
affected.

!!! note "What the minimum does not do"
    It stops a single meshblock's value being returned. It does not stop someone from inferring
    a value by differencing two regions that differ by one meshblock. Don't rely on a roll-up
    to hide the contribution of a small area.

## Area group rollup

```
GET /v1/rollups/area-group/{area_group_id}?indicator={key}&year={year}&variant={r|p|pl|pc}
```

Area groups are predefined by the AUO team; ask for the id of the group you need.

Averages the values of the group's member areas (an unweighted mean, rounded to the
indicator's precision), restricted to members within **your** effective scope and excluding
any member with no recorded value for that indicator/year/variant. A group's members are
[Areas](../concepts/areas.md) of any boundary type. Members aren't expanded —
a member that is an LGA contributes the LGA's own value, not its suburbs'.

### Response — 200

The body has these fields:

| Field | Type | Notes |
|---|---|---|
| `area_group_id` | integer | The group you asked for |
| `indicator` | string | As requested |
| `year` | integer | As requested |
| `variant` | string | As requested |
| `value` | number | The unweighted mean of the contributing members' values, rounded to the indicator's precision |

### Errors

1. `AREA_GROUP_NOT_FOUND` (404)
2. `INDICATOR_NOT_FOUND` (404)
3. `401`/`403` — you can't see the indicator at any area
4. `VARIANT_NOT_SUPPORTED` (422)
5. `INSUFFICIENT_DATA` (422) — the group has too few meshblock members ([minimum size](#minimum-size))
6. `401`/`403` — the group has members, but none are visible to you
7. `INSUFFICIENT_DATA` (422) — no visible member has a value for this indicator/year/variant

## Custom region rollup

A custom region is a persisted resource resolved once, at creation, to the set of meshblocks
fully contained within a submitted polygon (up to 5,000 vertices). Membership is resolved
once and stored; the **value** endpoint recomputes on every `GET`.

### Create a region

```
POST /v1/regions
```

```json
{
  "polygon": {
    "type": "Polygon",
    "coordinates": [
      [
        [144.975, -37.81],
        [144.99, -37.81],
        [144.99, -37.795],
        [144.975, -37.795],
        [144.975, -37.81]
      ]
    ]
  },
  "label": "Fitzroy test box"
}
```

```json
{ "id": 901, "url": "/v1/regions/901" }
```

`label` is optional. Creating a region doesn't need a key and doesn't read any values — the
access check happens when you ask for values. If the polygon resolves to zero meshblocks,
nothing is created. A region with fewer meshblocks than the [minimum](#minimum-size) *is*
created, but asking it for values is refused.

**Errors:** `CUSTOM_REGION_INVALID_GEOMETRY` (400) — malformed, self-intersecting, or over
5,000 vertices; `CUSTOM_REGION_EMPTY` (422) — valid polygon, zero meshblocks fully contained.

### Get a region's value for an indicator

```
GET /v1/regions/{region_id}/values?indicator={key}&year={year}&variant=r
```

Only `r` is ever supported here — see
[Variants → custom regions only ever support `r`](../concepts/variants.md#custom-regions-only-ever-support-r).
Computes the person-weighted mean of the region's meshblock-level values, live, on every call.
For example `GET /v1/regions/901/values?indicator=urban_liveability_index&year=2021&variant=r`:

```json
{
  "region_id": 901,
  "indicator": "urban_liveability_index",
  "year": 2021,
  "variant": "r",
  "value": 107.1
}
```

No meshblock count is returned, so you can't tell how many meshblocks a value rests on.

**Errors:**

1. `RESOURCE_NOT_FOUND` (404) — no region with that id
2. `INDICATOR_NOT_FOUND` (404)
3. `CUSTOM_REGION_NOT_SUPPORTED` (422) — indicator's `aggregation_method` is `unsupported`
4. `CUSTOM_REGION_VARIANT_NOT_SUPPORTED` (422) — anything other than `r`
5. `401 AUTHENTICATION_REQUIRED` / `403 OUT_OF_SCOPE` — you can't see the indicator anywhere
   for a roll-up
6. `INSUFFICIENT_DATA` (422) — the region has fewer than the [minimum](#minimum-size)
   meshblocks
7. `401`/`403` — your access doesn't cover any of the region's meshblocks for this indicator
8. `INSUFFICIENT_DATA` (422) — every member meshblock was excluded by scope, missing value, or
   missing weight (distinct from `CUSTOM_REGION_EMPTY`, which can only happen at creation —
   a region that exists always has at least one member)
