# Spatial

`spatial` — resolve coordinates and polygons against the area hierarchy. None of these three
endpoints apply effective-scope filtering — they resolve geometry, not indicator values. They
return the Areas they find as [AUO Area IDs](../concepts/areas.md) with names and source codes,
so they are also the way to get an ID from a location.

Every Area in a spatial response is described the same way:

```json
{ "area_id": "lga-000123", "name": "Yarra", "boundary_type": "lga",
  "source_codes": [{ "edition": "LGA 2021", "code": "27350" }] }
```

## Point-in-polygon

```
GET /v1/spatial/point?lat={lat}&lon={lon}&boundary_type={type}&year={year}
```

Returns the single area of that boundary type/year containing the coordinate.

```json
{
  "area": { "area_id": "lga-000123", "name": "Yarra", "boundary_type": "lga",
            "source_codes": [{ "edition": "LGA 2021", "code": "27350" }] },
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
  "area": { "area_id": "lga-000123", "name": "Yarra", "boundary_type": "lga",
            "source_codes": [{ "edition": "LGA 2021", "code": "27350" }] },
  "resolved_from": null,
  "year": 2021,
  "neighbours": [
    { "area_id": "lga-000044", "name": "Melbourne", "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "24600" }] },
    { "area_id": "lga-000310", "name": "Port Phillip", "boundary_type": "lga",
      "source_codes": [{ "edition": "LGA 2021", "code": "25900" }] }
  ]
}
```

`neighbours` is ordered by AUO Area ID. `resolved_from` is as in [Lookup](lookup.md#response-200).
Empty `neighbours: []` is a valid `200` if none exist.

**Errors:** `AREA_NOT_FOUND` (404) — an unknown AUO Area ID, a code that doesn't resolve (this
includes an unrecognised `boundary_type`), **or** a real Area that has no boundary in `year`
(then `details` carries the Area and the `year`); `422` for an Area named both ways or neither
way ([malformed request](../concepts/errors.md#malformed-requests)).

## Polygon containment

```
POST /v1/spatial/contains
```

```json
{ "polygon": { "type": "Polygon", "coordinates": [[...]] }, "boundary_type": "sa1", "year": 2021 }
```

Returns every area of the given boundary type/year whose geometry lies **entirely within**
the submitted polygon.

```json
{
  "boundary_type": "sa1",
  "year": 2021,
  "areas": [
    { "area_id": "sa1-001234", "name": null, "boundary_type": "sa1",
      "source_codes": [{ "edition": "SA1 2021", "code": "10123456" }] },
    { "area_id": "sa1-001235", "name": null, "boundary_type": "sa1",
      "source_codes": [{ "edition": "SA1 2021", "code": "10123457" }] }
  ]
}
```

`areas` is ordered by AUO Area ID. SA1s have no name, so `name` is `null`. Not a persisted
resource — no corresponding `GET /{id}`. **Errors:** `INVALID_GEOMETRY` (400) — checked first,
no database access needed; `UNKNOWN_BOUNDARY_TYPE_OR_YEAR` (400).
