# Admin

`admin` — grants, area groups, coverage gaps, and API key management. Every endpoint on this
page requires an **admin-scoped key**, not a regular grantee key — checked first, before
anything else runs.

| Situation | Response |
|---|---|
| No bearer token | `401 AUTHENTICATION_REQUIRED` |
| Valid key, but not admin-scoped | `403 OUT_OF_SCOPE` |

## Get a grantee's effective scope

```
GET /v1/admin/grantees/{grantee}/scope
```

```json
{ "indicator_ids": [4, 17, 42], "area_codes": ["20660", "20140"] }
```

Empty arrays if the grantee exists but currently holds no active grants. `area_codes` is
deduplicated across boundary types — it doesn't distinguish which boundary type each code
belongs to. **Errors:** `GRANTEE_NOT_FOUND` (404).

## Coverage gaps

```
GET /v1/admin/coverage?boundary_type={type}&year={year}
```

Every indicator with zero recorded values for that boundary type and year:

```json
[{ "key": "walkability" }, { "key": "public_transport_access" }]
```

Empty list if the boundary type/year exist but have no gaps. **Errors:**
`UNKNOWN_BOUNDARY_TYPE_OR_YEAR` (**404** here — note this is a different status than the
same code returns from the spatial endpoints, which use 400).

## Area group members

```
GET /v1/admin/area-groups/{area_group_id}/members
```

```json
[{ "area_identifier": "20660", "include_child_areas": true }]
```

**Errors:** `AREA_GROUP_NOT_FOUND` (404).

## Issue an API key

```
POST /v1/admin/grantees/{grantee}/keys
```

```json
{ "label": "Partner integration key" }
```

```json
{ "id": 55, "key": "sk_live_...", "label": "Partner integration key" }
```

The plaintext `key` is returned **exactly once, here** — it is never retrievable again after
this response. `label` is optional. A new key attaches to the grantee's most recently
created grant. **Errors:** `GRANTEE_NOT_FOUND` (404).

## Revoke an API key

```
DELETE /v1/admin/keys/{key_id}
```

```json
{ "id": 55, "revoked_at": "2026-01-15T09:32:00Z" }
```

Idempotent — revoking an already-revoked key returns the original revocation timestamp
unchanged, not an error. Every request subsequently authenticated with a revoked key gets
`401 INVALID_API_KEY`. **Errors:** `RESOURCE_NOT_FOUND` (404) — no key with that id at all.
