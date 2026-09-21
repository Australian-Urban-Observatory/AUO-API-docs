# Spatial

`spatial` — resolve coordinates and polygons against the area hierarchy. None of these three
endpoints apply effective-scope filtering — they resolve geometry, not indicator values.

## Point-in-polygon

```
GET /v1/spatial/point?lat={lat}&lon={lon}&boundary_type={type}&year={year}
```

Returns the single area of that boundary type/year containing the coordinate.

```json
{ "community_code": "20660", "boundary_type": "lga", "year": 2021 }
```

**Errors:** `INVALID_GEOMETRY` (400) — lat outside ±90 or lon outside ±180, checked before any
database access; `UNKNOWN_BOUNDARY_TYPE_OR_YEAR` (400); `NO_CONTAINING_AREA` (404) — valid
coordinate, but no area of that boundary type/year contains it.

## Neighbours

```
GET /v1/areas/{community_code}/neighbours?boundary_type={type}&year={year}
```

Returns every other area of the same boundary type/year whose geometry shares a boundary
edge with the given area's.

```json
{
  "community_code": "20660",
  "boundary_type": "lga",
  "year": 2021,
  "neighbours": ["20140", "20310", "20450"]
}
```

Empty `neighbours: []` is a valid `200` if none exist. **Errors:** `AREA_NOT_FOUND` (404) —
covers both an unknown `community_code` and an unresolvable `boundary_type`.

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
{ "boundary_type": "sa1", "year": 2021, "areas": ["10123456", "10123457"] }
```

Not a persisted resource — no corresponding `GET /{id}`. **Errors:** `INVALID_GEOMETRY` (400)
— checked first, no database access needed; `UNKNOWN_BOUNDARY_TYPE_OR_YEAR` (400).
