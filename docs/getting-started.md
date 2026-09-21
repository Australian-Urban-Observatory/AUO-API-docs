# Getting Started

!!! warning "No public deployment yet"
    The AUO Data API is not yet deployed to a public base URL. Everything below applies once
    it is; until then, run it locally per `AUO-API`'s own README (`uvicorn app.main:app`) and
    substitute `http://localhost:8000` for the base URL in every example in this manual.

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

Most of the catalogue — indicators and areas that aren't partner-restricted — is readable
with **no API key at all**. You only need a key for:

- Partner-restricted indicators (`partner_indicator = true` in the catalogue)
- Areas your organisation has been granted narrower-than-public access to
- Every `/v1/admin/...` endpoint

Send your key as a bearer token:

```
Authorization: Bearer <your-api-key>
```

| Situation | Status | Error code |
|---|---|---|
| No token, and the request needs one | 401 | `AUTHENTICATION_REQUIRED` |
| Token present but doesn't match a valid, non-revoked key | 401 | `INVALID_API_KEY` |
| Token valid, but the request targets something outside your grant | 403 | `OUT_OF_SCOPE` |

**There is currently no self-service way to get a key.** Key issuance is an admin action
(`POST /v1/admin/grantees/{grantee}/keys`) performed by the AUO team on request — contact the
AUO team directly to arrange access.

Your key is tied to your organisation ("grantee"), not to an individual user, and it
authorises whatever set of indicators and areas your organisation's access grant covers (its
**effective scope** — see [Scope, grants & auth](concepts/scope-and-auth.md)). A key is shown
in plaintext exactly once, at issuance; if you lose it, ask the AUO team to revoke it and
issue a new one.

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

No `Authorization` header was needed here because `walkability` and this area are both
publicly readable. See [API Reference → Lookup a value](reference/lookup.md) for the full
parameter and error reference for this endpoint.
