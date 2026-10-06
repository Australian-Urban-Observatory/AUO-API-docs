# Getting Started

!!! warning "No public deployment yet"
    The AUO Data API is not yet deployed to a public base URL. Everything below applies once
    it is; contact the AUO team for the current base URL and substitute it for `{base_url}`
    in every example in this manual.

## Base URL

Every endpoint is versioned under `/v1`:

```
{base_url}/v1/...
```

## Versioning

A version prefix, once released, is a stable contract: existing fields and status codes
never change meaning within `/v1`. New, optional fields may be added without notice. A
breaking change ships under a new prefix (`/v2`) rather than changing `/v1` underneath you.

Requesting a prefix that doesn't exist, or one that has been withdrawn, is rejected before
your request reaches any handler:

| Situation | Status | Error code |
|---|---|---|
| Prefix never released (e.g. `/v2` today) | 404 | `UNSUPPORTED_VERSION` |
| Prefix withdrawn | 410 | `VERSION_WITHDRAWN` |

## Authentication

Part of the catalogue is readable with **no credential at all**: every indicator that isn't
partner-restricted, at state, study-region and LGA level. You need a credential for:

- Partner-restricted indicators (`partner_indicator = true` in the catalogue)
- Suburb (`ssc`) and `sa1` data — public access stops at LGA level
- Roll-ups over meshblocks, such as [custom regions](reference/rollup.md#custom-region-rollup)

What a credential unlocks is decided by your organisation's access — see
[Scope, grants & auth](concepts/scope-and-auth.md).

Send your credential as a bearer token:

```
Authorization: Bearer <your-api-key>
```

The token is normally an **API key**. The API also accepts an **access token from AUO
sign-in**, which is what AUO's own applications send on behalf of a signed-in person; you'll
recognise one by its shape (three dot-separated parts). A signed-in person is linked to their
organisation by the AUO team; someone who signs in but hasn't been linked gets public access.

| Situation | Status | Error code |
|---|---|---|
| No token, and the request needs one | 401 | `AUTHENTICATION_REQUIRED` |
| API key doesn't match a valid, non-revoked key | 401 | `INVALID_API_KEY` |
| Sign-in token is malformed, wrongly signed or expired | 401 | `INVALID_TOKEN` (`details.reason` is `invalid` or `expired`) |
| Credential is valid, but doesn't cover what was requested | 403 | `OUT_OF_SCOPE` |

An expired sign-in token is **never** quietly treated as "no token": you get `INVALID_TOKEN`
with `reason: "expired"` so you can refresh it and retry, rather than seeing data disappear.

**There is currently no self-service way to get a key.** Contact the AUO team to arrange
access.

Your key is tied to your organisation, not to an individual user, and it carries whatever
access your organisation has been granted (its **effective scope**). An organisation has one
active key at a time: a key is shown in plaintext exactly once, when it's issued, and if you
lose it the AUO team can replace it (which revokes the old one).

## Making your first request

```bash
curl "{base_url}/v1/values?community_code=20660&boundary_type=lga&indicator=walkability&year=2021"
```

```json
{
  "community_code": "20660",
  "boundary_type": "lga",
  "indicator": "walkability",
  "year": 2021,
  "variant": "r",
  "value": 4.2,
  "polarity": "positive",
  "units": "index score"
}
```

No `Authorization` header was needed here because `walkability` is a non-restricted
indicator and this area is at LGA level. See [API Reference → Lookup a
value](reference/lookup.md) for the full parameter and error reference for this endpoint.

Don't know the `community_code` for a place? [Search areas by name](reference/search.md#search-areas).
