# Areas & AUO Area IDs

An **Area** is a place AUO reports on: a state, study region, LGA, suburb, SA1 (or, behind the
scenes, a meshblock). Every endpoint that names a place names an Area, and the one identifier
you should store and send back is its **AUO Area ID**.

## AUO Area ID

```
lga-000123
```

- **Permanent.** An ID is never reused for a different place and never reassigned. An Area
  keeps its ID even after a later data release no longer contains it.
- **Opaque.** Today an ID reads `<boundary type>-<number>`, so you can tell the kind at a glance,
  but treat it as an opaque string: don't parse it, build it, or rely on the format.
- **Not an ABS code.** An ABS code is only meaningful inside one ABS edition. Suburb codes in
  particular were numbered independently by the 2016 (SSC) and 2021 (SAL) editions, so the same
  code can be two different places. An AUO Area ID is the same place in every release it appears
  in.

IDs in the examples throughout this manual are illustrative.

## What an Area looks like in a response

Every response that mentions an Area describes it the same way:

```json
{
  "area_id": "ssc-000812",
  "name": "Brunswick",
  "boundary_type": "ssc",
  "source_codes": [
    { "edition": "SAL 2021", "code": "20141" },
    { "edition": "SSC 2016", "code": "20204" }
  ]
}
```

- `name` is the Area's name in its **newest** year. SA1s and meshblocks have no name, so `name`
  is `null` for them.
- `source_codes` lists every ABS code the Area is known by, one per ABS edition, newest edition
  first. Use it to join AUO results to your own ABS-keyed data.

## Naming an Area in a request

Single-area endpoints ([lookup](../reference/lookup.md), [profile](../reference/profile.md),
[trend](../reference/trend.md), [neighbours](../reference/spatial.md#neighbours),
[compare indicators](../reference/compare-indicators.md)) accept either form:

| Form | Parameters |
|---|---|
| **AUO Area ID** (preferred) | `area=lga-000123` |
| **ABS code** | `community_code=27350&boundary_type=lga`, optionally `&edition=LGA 2021` |

Give one form or the other, never both: mixing them, or leaving both out, is a `422` (see
[malformed requests](errors.md#malformed-requests)).

A bare ABS code is ambiguous across editions, so the API resolves it for you:

- With `edition`, the code is looked up in exactly that edition.
- Without it, in the **newest** edition that contains the code.

The response always says which Area it landed on (`area`) and, for a code, which source code
matched (`resolved_from`; it is `null` when you sent an AUO Area ID). If you started from a code,
keep the `area.area_id` from the response and use it from then on.

A reference that matches nothing is `404 AREA_NOT_FOUND`.

Endpoints that take lists of Areas ([compare areas](../reference/compare-areas.md)) or a scope
(`scope_code` on [rank](../reference/rank.md), [distribution](../reference/distribution.md) and
[correlate](../reference/correlate.md)) take AUO Area IDs; compare areas also accepts codes.

### Where to get an ID

- [Search areas](../reference/search.md#search-areas) — from a place name.
- [Point-in-polygon](../reference/spatial.md#point-in-polygon) — from a coordinate.
- [Polygon containment](../reference/spatial.md#polygon-containment) and
  [neighbours](../reference/spatial.md#neighbours) — from geometry and adjacency.
- [Rankings](../reference/rank.md) — every entry carries its `area_id`.

## Areas across years

An Area has a **feature** (a name and a boundary) in each year it exists. Names and boundaries
are per year: an Area is labelled with its newest year's name, while
[area search](../reference/search.md#search-areas) also matches the names it had earlier, so a
renamed suburb is found by its old name too.

Whether a place in one release is the same Area as a place in another is decided once, when
the data is loaded, by a rule for each boundary type:

| Boundary type | Same Area across ABS editions when… |
|---|---|
| state, study region, LGA, SA1, meshblock | the ABS code is the same |
| suburb (`ssc`) | the 2016 and 2021 boundaries overlap closely (intersection-over-union of at least 0.8, one-to-one) |

For suburbs this means a place keeps one Area, and so one [trend](../reference/trend.md), across
the 2016 → 2021 renumbering. A small share of suburbs (splits, merges, suburbs that moved
study region) do not match and become Areas of their own: an older Area with only 2018 data
and a newer one with 2021 and 2024. A matched suburb's 2018 value was computed on its 2016
boundary, the same approximation already made for LGAs whose boundaries moved.

### Merged Areas and predecessors

When two Areas merged into one (Rockdale and Botany Bay became Bayside (NSW); Mallala became part
of Adelaide Plains), the later Area lists them as its **predecessors** on its
[trend](../reference/trend.md#predecessors). Their values are never combined into the later
Area's, and the later Area gets no invented earlier value. The predecessors stay Areas in their
own right, with their own IDs and values.

## Assigned parents

Each Area at LGA level or below has, per year, at most one **assigned parent** at each coarser
boundary type: its state, study region, LGA and (for SA1s) suburb. The assignment is the data
source's own, and it is what the percentile variants and the `scope` of a ranking are built on.

It is **not** geometric containment. Finer areas don't nest exactly: an SA1 can straddle two
LGAs yet has one assigned LGA, and an LGA can be assigned to a study region it doesn't lie
wholly inside. Things that depend on it:

- A [ranking, distribution or correlation](../reference/rank.md#scopes) scoped to a state,
  study region or LGA covers the Areas that have that scope Area as their assigned parent in
  the requested year.
- The `context` label on [area search](../reference/search.md#search-areas) results.
- **2018 Areas have no assigned parents**, so they belong to no state, study-region or LGA
  scope and have no `context`. A scoped query for 2018 finds nothing; a `national` one still
  works. Their `pl` and `pc` values are still served — see [Variants](variants.md).

For geometric questions ("which SA1s lie inside this polygon") use the
[spatial endpoints](../reference/spatial.md), which test geometry directly.
