# Search

Two search endpoints resolve plain language to something you can query with:

- [`GET /v1/search/indicators`](#search-indicators) — a description → candidate indicators
- [`GET /v1/search/areas`](#search-areas) — a place name → candidate areas

Both take a single query string `q` (1–200 characters), return `candidates` ordered by
descending `score`, and only consider what **your effective scope** lets you see. A `q` that
is empty or over 200 characters is `400 QUERY_INVALID`; a `q` that is valid but matches
nothing is a normal `200` with `no_match: true`.

## Search indicators

`search` — find an indicator from a plain-language description, without knowing its key.

```
GET /v1/search/indicators?q={free text}
```

### Response — 200

```json
{
  "query": "how easy it is to walk to shops",
  "no_match": false,
  "candidates": [
    { "key": "walkability", "score": 1.0, "reason": "matched “walk”, “shops”" },
    { "key": "daily_living_access", "score": 0.5, "reason": "matched “shops”" }
  ]
}
```

- `candidates` is capped at 20, ordered by descending `score`, ties broken by ascending key.
- `score` is relative to the best match for *this* query: the top candidate is always `1.0`.
  It ranks the candidates against each other; it isn't a confidence level you can compare
  across queries.
- `no_match: true` with `candidates: []` is a valid `200` — a query that matches nothing is
  not an error.
- Results are already restricted to indicators you can see at one or more areas — a
  partner-restricted indicator you have no access to never appears as a candidate.

### Declined terms

Some terms are explicitly outside what the AUO catalogue holds (cycling infrastructure,
gambling data, crime data, etc.) and are declined rather than silently matched to something
adjacent:

```json
{
  "query": "show me the pokies data",
  "no_match": true,
  "declined": [
    { "term": "pokies", "reason": "We hold alcohol outlet density, but no gambling or gaming machine data." }
  ],
  "candidates": []
}
```

`declined` is **omitted entirely** (not an empty array) when nothing in the query matched
the refusal list. A declined term is never itself scored as a candidate, but the rest of the
query is still scored normally — a query can carry both `declined` entries and real
`candidates` at once. `no_match` only reflects whether `candidates` is empty, so check
`declined` for the reason.

## Search areas

`search-areas` — find an area from a place name, when you don't already have its
`community_code`.

```
GET /v1/search/areas?q={place name}
```

This is a fallback for a client with no area selected yet. To find the area containing a
coordinate, use [Spatial → point-in-polygon](spatial.md#point-in-polygon) instead.

### Response — 200

Codes in this example are illustrative.

```json
{
  "query": "my LGA is Yarra",
  "no_match": false,
  "candidates": [
    {
      "community_code": "27350",
      "boundary_type": "lga",
      "name": "Yarra",
      "context": "VIC",
      "years": [2018, 2021, 2024],
      "score": 1.0,
      "reason": "matched “yarra”"
    },
    {
      "community_code": "21310",
      "boundary_type": "lga",
      "name": "Yarra Ranges",
      "context": "VIC",
      "years": [2018, 2021, 2024],
      "score": 0.5,
      "reason": "matched “yarra”"
    }
  ]
}
```

- `boundary_type` is one of `state`, `study_region`, `lga`, `ssc`. SA1s and meshblocks have
  no names, so they're never returned.
- `name` is the area's name. `context` locates it for a human — the state for an LGA, the
  LGA and state for a suburb — and is `null` for a state or study region.
- `years` lists the releases the area exists in. The `community_code` is the same across
  them, but see [Boundary hierarchy](../concepts/boundary-hierarchy.md) for why codes can
  change between releases.
- Matching ignores case and punctuation (`Merri-bek`, `merri bek` and `MERRI-BEK` are the
  same). A query word matches a name word exactly, or — if the query word is 3 or more
  characters — as a prefix. One-character words are ignored, as are words that match nothing
  (so filler like "my" and "is" costs nothing).
- A full-name match scores higher than a partial one, so `yarra` ranks *Yarra* above *Yarra
  Ranges*. Including a boundary word — `lga`, `council`, `suburb`, `state` and similar —
  prefers that kind of area over a same-named one of another kind.
- Ordered by descending `score`, then coarsest boundary type first, then name, then
  `community_code`; capped at 20. `score` is relative to the best match for this query, so
  the top candidate is `1.0`.
- There is no `declined` field: a topic belongs in indicator search, not here.

### What you can find

Only areas where your effective scope lets you see at least one indicator are candidates.
That is every state, study region and LGA for everyone — but a **suburb only appears if your
access covers it**. An unauthenticated search for a suburb name returns `no_match: true`.

### Errors

`QUERY_INVALID` (400) — `q` is empty or over 200 characters. A query that reduces to no
usable words (all punctuation, or single characters) is **not** an error — it's a `no_match:
true` result.
