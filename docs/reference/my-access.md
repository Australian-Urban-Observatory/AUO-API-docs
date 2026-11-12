# My access

`my-access` — what **you** can see, per boundary type, so an app can grey out the scales and
indicators you can't use before asking for them. It works with no credential (the public
baseline), an API key, or a signed-in person's token.

```
GET /v1/me/access
```

No parameters.

## Response — 200

Without a credential:

```json
{
  "authenticated": false,
  "grantee": false,
  "boundary_types": [
    { "boundary_type": "state", "name": "State", "access": "all_areas",
      "all_areas": { "all_public": true, "indicators": [] }, "some_areas": [] },
    { "boundary_type": "study_region", "name": "Study Region", "access": "all_areas",
      "all_areas": { "all_public": true, "indicators": [] }, "some_areas": [] },
    { "boundary_type": "lga", "name": "Local Government Area", "access": "all_areas",
      "all_areas": { "all_public": true, "indicators": [] }, "some_areas": [] },
    { "boundary_type": "ssc", "name": "State Suburb", "access": "none",
      "all_areas": { "all_public": false, "indicators": [] }, "some_areas": [] },
    { "boundary_type": "sa1", "name": "Statistical Area 1", "access": "none",
      "all_areas": { "all_public": false, "indicators": [] }, "some_areas": [] }
  ]
}
```

For an organisation whose access covers one partner-restricted indicator at two suburbs
(trimmed to the `ssc` entry):

```json
{
  "authenticated": true,
  "grantee": true,
  "boundary_types": [
    {
      "boundary_type": "ssc",
      "name": "State Suburb",
      "access": "some_areas",
      "all_areas": { "all_public": false, "indicators": [] },
      "some_areas": [
        { "indicators": ["walkability_index"], "area_ids": ["ssc-000619", "ssc-001008"] }
      ]
    }
  ]
}
```

A full partner sees `"access": "all_areas"`, `"all_public": true` and every partner-restricted
indicator key in `all_areas.indicators` at every boundary type.

- `authenticated` is `true` when you sent a credential and it was accepted. `grantee` is `true`
  when it acts for an organisation (an API key, or a person linked to one). A signed-in person
  who isn't linked to an organisation gets `authenticated: true`, `grantee: false` and the public
  baseline.
- `boundary_types` lists every boundary type you could map, coarsest first. Meshblock is never
  listed: meshblock values are never returned directly
  ([Scope, grants & auth](../concepts/scope-and-auth.md#meshblocks-are-roll-up-only)).
- `all_areas` is what you can see at **every** area of that boundary type. `all_public: true`
  means every public (not partner-restricted) indicator; `indicators` lists the keys of any
  other indicator visible everywhere there.
- `some_areas` is what you can see only at particular areas, as groups: each group's
  `indicators` are visible at exactly its `area_ids` ([AUO Area IDs](../concepts/areas.md)).
  Indicators already in `all_areas` aren't repeated. Groups are ordered by their first indicator
  key; keys and area IDs are sorted within a group.
- `access` sums it up: `all_areas` if anything is visible at every area, otherwise `some_areas`
  if anything is visible at some, otherwise `none`.

This is a summary of the same access every other endpoint applies, read fresh on every request;
it never grants anything by itself. The indicator catalogue's
[`availability`](metadata.md) says where data *exists*; this page says what you can *see*. An
indicator you can see at some boundary type and year still needs data there.

The response depends on who asks and is sent with `Cache-Control: private, no-store`.

## Errors

- `401 INVALID_API_KEY` / `401 INVALID_TOKEN` — a credential was sent but isn't valid (as on
  every endpoint). Send no `Authorization` header to get the public baseline.

Also the usual [version-prefix errors](../getting-started.md#versioning).
