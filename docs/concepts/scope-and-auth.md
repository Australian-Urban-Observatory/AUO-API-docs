# Scope, grants & auth

## Grantee, grant, effective scope

- A **grantee** is the organisation your API key belongs to. Grants are resolved at the
  grantee level — every key issued to your organisation shares the same access.
- A **grant** (`access_grants` record) authorises visibility of a set of indicators (by
  indicator or domain) and a set of areas (by area identifier or area group), optionally
  restricted to specific boundary types.
- Your **effective scope** is the resolved set of indicator ids and area codes your grant
  expands to — this is what every request is actually checked against, not the raw grant
  record.

Public (unauthenticated) requests have an effective scope too: every indicator that isn't
partner-restricted, and every area not named by any access grant.

## Two different ways scope gets enforced

This matters because it changes what kind of response you get when you ask for something
partly outside your scope:

**Rejecting** — naming something specific that's out of scope fails the whole request with
`403 OUT_OF_SCOPE`. Applies to endpoints where you name one specific indicator or area
directly: [Lookup](../reference/lookup.md), [Trend](../reference/trend.md),
[indicator metadata by key](../reference/metadata.md).

**Filtering** — aggregate/multi-area endpoints silently exclude out-of-scope areas from the
computation rather than rejecting the request. Applies to
[Profile](../reference/profile.md), [Rank](../reference/rank.md),
[Distribution](../reference/distribution.md), [Correlate](../reference/correlate.md), and
[Rollup](../reference/rollup.md). If effective-scope filtering removes every contributing
area, you get `422 INSUFFICIENT_DATA`, not an empty success — and count-based fields (rank
totals, distribution counts) reflect the filtered set, not what you originally asked for.

## Partner-restricted indicators

An indicator with `partner_indicator = true` is only visible to a grantee whose effective
scope includes it. An unauthenticated request to the full catalogue
(`GET /v1/indicators`) silently omits these; a direct request naming one by key
(`GET /v1/indicators/{key}`) without the right grant returns `403 OUT_OF_SCOPE`.

## Computed resources and scope

Comparisons, correlations, and custom regions are persisted resources with their own URL
(`POST` creates, `GET /v1/.../{id}` retrieves). The **creator's** scope only matters at
creation for the `result` field returned inline with the `201` — every subsequent `GET`
re-resolves against **that caller's own** effective scope, intersected with the resource's
stored inputs. If your scope doesn't overlap the resource's inputs at all, you get
`403 OUT_OF_SCOPE` on the `GET`, even if the resource itself is real and someone else can see
it fine.

## Cache invalidation

Effective scope is cached server-side per grant. Creating, modifying, or revoking a grant,
API key, or area group invalidates the cache immediately — you don't need to do anything
client-side, but don't assume a permission change is instant if you're hammering the API in
a tight loop right at the moment of the change.
