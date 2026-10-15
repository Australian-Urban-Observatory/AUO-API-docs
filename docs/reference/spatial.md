# Spatial

`spatial` — resolve coordinates and polygons against the area hierarchy. None of these three
endpoints apply effective-scope filtering — they resolve geometry, not indicator values. They
return the Areas they find as [AUO Area IDs](../concepts/areas.md) with names and source codes,
so they are also the way to get an ID from a location.

Every Area in a spatial response is described the same way:

```json
{
  "area_id": "lga-000084",
  "name": "Yarra",
  "boundary_type": "lga",
  "source_codes": [{ "edition": "LGA 2021", "code": "27350" }, { "edition": "LGA 2016", "code": "27350" }]
}
```

## Point-in-polygon

```
GET /v1/spatial/point?lat={lat}&lon={lon}&boundary_type={type}&year={year}
```

Returns the single area of that boundary type/year containing the coordinate. For example
`GET /v1/spatial/point?lat=-37.8&lon=144.98&boundary_type=lga&year=2021`:

```json
{
  "area": {
    "area_id": "lga-000084",
    "name": "Yarra",
    "boundary_type": "lga",
    "source_codes": [{ "edition": "LGA 2021", "code": "27350" }, { "edition": "LGA 2016", "code": "27350" }]
  },
  "boundary_type": "lga",
  "year": 2021
}
```

`boundary_type` and `year` echo the request; `area` is the Area that contains the point **in that
year's boundary**.

**Errors:** `INVALID_GEOMETRY` (400) — lat outside ±90 or lon outside ±180, checked before any
database access; `UNKNOWN_BOUNDARY_TYPE_OR_YEAR` (400); `NO_CONTAINING_AREA` (404) — valid
coordinate, but no area of that boundary type/year contains it.

## Neighbours

```
GET /v1/areas/neighbours?area={AUO Area ID}&year={year}
GET /v1/areas/neighbours?community_code={code}&boundary_type={type}[&edition={edition}]&year={year}
```

Returns every other area of the same boundary type whose geometry in that year shares a
boundary edge with the given area's. The Area is named as in [Lookup](lookup.md#query-parameters)
— by `area`, or by `community_code` with `boundary_type` (and optionally `edition`) — and is
no longer part of the URL path.

```json
{
  "area": {
    "area_id": "lga-000084",
    "name": "Yarra",
    "boundary_type": "lga",
    "source_codes": [{ "edition": "LGA 2021", "code": "27350" }, { "edition": "LGA 2016", "code": "27350" }]
  },
  "resolved_from": null,
  "year": 2021,
  "neighbours": [
    {
      "area_id": "lga-000047",
      "name": "Banyule",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "20660" }, { "edition": "LGA 2016", "code": "20660" }]
    },
    {
      "area_id": "lga-000049",
      "name": "Boroondara",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "21110" }, { "edition": "LGA 2016", "code": "21110" }]
    },
    {
      "area_id": "lga-000053",
      "name": "Darebin",
      "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "21890" }, { "edition": "LGA 2016", "code": "21890" }]
    }
  ]
}
```

`neighbours` is ordered by AUO Area ID. `resolved_from` is as in [Lookup](lookup.md#response-200).
The example is `GET /v1/areas/neighbours?area=lga-000084&year=2021`, trimmed to the first three
of Yarra's six neighbours. Empty `neighbours: []` is a valid `200` if none exist.

**Errors:** `AREA_NOT_FOUND` (404) — an unknown AUO Area ID, a code that doesn't resolve (this
includes an unrecognised `boundary_type`), **or** a real Area that has no boundary in `year`
(then `details` carries the Area and the `year`); `422` for an Area named both ways or neither
way ([malformed request](../concepts/errors.md#malformed-requests)).

## Polygon containment

```
POST /v1/spatial/contains
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
  "boundary_type": "sa1",
  "year": 2021
}
```

Returns every area of the given boundary type/year whose geometry lies **entirely within**
the submitted polygon. The request above is a box over Fitzroy, and the response is trimmed to the
first two of the 30 SA1s the call returns.

```json
{
  "boundary_type": "sa1",
  "year": 2021,
  "areas": [
    {
      "area_id": "sa1-014558",
      "name": null,
      "boundary_type": "sa1",
      "source_codes": [
        { "edition": "SA1 2021", "code": "20607114108" },
        { "edition": "SA1 2016", "code": "20607114108" }
      ]
    },
    {
      "area_id": "sa1-014561",
      "name": null,
      "boundary_type": "sa1",
      "source_codes": [
        { "edition": "SA1 2021", "code": "20607114111" },
        { "edition": "SA1 2016", "code": "20607114111" }
      ]
    }
  ]
}
```

`areas` is ordered by AUO Area ID. SA1s have no name, so `name` is `null`. Not a persisted
resource — no corresponding `GET /{id}`. **Errors:** `INVALID_GEOMETRY` (400) — checked first,
no database access needed; `UNKNOWN_BOUNDARY_TYPE_OR_YEAR` (400).
