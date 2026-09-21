# Search

`search` — find an indicator from a plain-language description, without knowing its key.

```
GET /v1/search/indicators?q={free text}
```

## Response — 200

```json
{
  "query": "how easy it is to walk to shops",
  "no_match": false,
  "candidates": [
    { "key": "walkability", "score": 0.83, "reason": "matched 'walk' in description" },
    { "key": "daily_living_access", "score": 0.41, "reason": "matched 'shops' in domain" }
  ]
}
```

- `candidates` is capped at 20, ordered by descending `score` (0.0–1.0), ties broken by
  ascending key.
- `no_match: true` with `candidates: []` is a valid `200` — a query that matches nothing is
  not an error.
- Results are already restricted to your effective scope — a partner-restricted indicator
  you can't see never appears as a candidate.

## Declined terms

Some terms are explicitly outside what the AUO catalogue holds (cycling infrastructure,
gambling data, crime data, etc.) and are declined rather than silently matched to something
adjacent:

```json
{
  "query": "show me the pokies data",
  "no_match": false,
  "declined": [
    { "term": "pokies", "reason": "We hold off-licence alcohol outlet density, not gambling data." }
  ],
  "candidates": []
}
```

`declined` is **omitted entirely** (not an empty array) when nothing in the query matched
the refusal list. A declined term is never itself scored as a candidate, but the rest of the
query is still scored normally — a query can carry both `declined` entries and real
`candidates` at once.

## Errors

`QUERY_INVALID` (400) — `q` is empty or over 200 characters. A query that tokenizes to
nothing useful (e.g. all punctuation) is **not** an error — it's a legitimate `no_match: true`
result.
