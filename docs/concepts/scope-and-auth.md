# Scope, grants & auth

What you can read is decided by your **effective scope**: a set of *(indicator, area)*
combinations. This page explains how that set is built and what the API does when a
request reaches outside it.

## The public baseline

With no API key, you can read:

> every indicator that isn't partner-restricted, at every area of **LGA level or coarser**
> (`state`, `study_region`, `lga`).

Anything else needs a key:

| You want | Needs a key? |
|---|---|
| A non-restricted indicator at state, study-region or LGA level | No |
| A non-restricted indicator at **suburb (`ssc`) or `sa1`** level | Yes — public access stops at LGA |
| A partner-restricted indicator (`partner_indicator = true`) at any level | Yes |
| A `meshblock` value | Never readable directly — see [Meshblocks](#meshblocks-are-roll-up-only) |

## Grantee, grant, effective scope

- A **grantee** is the organisation your API key belongs to. An organisation has at most
  one active key: issuing a new one revokes the previous one.
- A **grant** is given to a grantee and lists exact indicators and exact areas. It covers
  **every combination of one of its indicators with one of its areas, and nothing else.**
    - An area is an exact [AUO Area ID](areas.md). Naming a suburb does not grant the SA1s
      inside it, or the LGA around it, and an ABS code is never what a grant names — so a
      suburb code that a later ABS edition reuses for a different place does not widen a
      grant.
    - It applies to that Area in every year it has a feature.
    - A grant never takes anything away from the public baseline.
- Your **effective scope** is the public baseline plus everything covered by every grant
  your organisation holds. It belongs to the organisation, never to a particular key.

Because combinations are explicit, a grant for a restricted indicator at your own SA1s does
not let you read it anywhere else, and a grant for an SA1 only unlocks the indicators it
lists there, not every public indicator at that SA1.

Some organisations hold a broad, standing grant — every indicator at every boundary type.
That kind of grant follows the data: an indicator or area loaded later is covered without
anyone editing anything. A narrow grant never changes unless its terms do.

To find out which Areas a narrow grant covers, call the endpoints and read the results (for
example [area search](../reference/search.md#search-areas) lists only the places you can see),
or ask the AUO team. Because a grant names Areas by AUO Area ID, your own copy of any list of
covered Areas should be keyed by AUO Area ID, not ABS code.

Your scope is worked out **live on every request**. There's no delay after access is
granted, narrowed or revoked: the next request sees the change.

## How requests are checked

Every endpoint applies the same ideas, in this order:

1. **Things that don't exist are `404`s.** An unknown area, indicator or scope is reported
   as not found before any access check — you will never get a `401`/`403` for something
   that doesn't exist.
2. **Can you see the indicator anywhere at all?** If not, the request is rejected.
3. **Is this exact combination covered?** Endpoints that name one area and one indicator
   reject if not. Endpoints that cover many areas instead *filter* — see below.

A rejection is one of two codes, depending on whether you sent a credential:

| Situation | Status | Code |
|---|---|---|
| No credential, and the request needs one | 401 | `AUTHENTICATION_REQUIRED` |
| Credential presented, but it doesn't cover the request | 403 | `OUT_OF_SCOPE` |

Both carry `details.resource_type` and `details.identifier` naming what was refused. A
restricted indicator's *key* is not secret — a `401`/`403` confirms it exists — but its
metadata and values are.

### Rejecting endpoints

These name one specific indicator and/or area, so anything outside your scope fails the
whole request:

- [Lookup](../reference/lookup.md) and [Trend](../reference/trend.md) — the indicator must
  be visible somewhere, *and* the exact indicator/area combination must be covered.
- [Area profile](../reference/profile.md) — the area must have at least one indicator
  visible at it.
- [Indicator metadata by key](../reference/metadata.md).

### Filtering endpoints

Multi-area endpoints silently drop the areas you can't see and compute over the rest:
[Rank](../reference/rank.md), [Distribution](../reference/distribution.md),
[Correlate](../reference/correlate.md), [Compare areas](../reference/compare-areas.md),
[Compare indicators](../reference/compare-indicators.md), and the
[roll-ups](../reference/rollup.md). Counts in the response (`total`, `count`)
describe the filtered set, not what you asked for.

Two consequences worth knowing:

- **Filtering that removes everything is a rejection, not an empty result.** If data exists
  but none of it is visible to you, you get `401`/`403`, not `200` with an empty list or
  `422 INSUFFICIENT_DATA`. A result that is empty because nothing is recorded at all
  stays `200` (or `INSUFFICIENT_DATA`), so "nothing there" and "nothing you can see" are
  told apart. If you can see *some* of the data, but too little to compute a result, you
  get `INSUFFICIENT_DATA`.
- **Partial visibility is silent.** If you can see some suburbs in an LGA but not others,
  a ranking of that LGA's suburbs ranks only the ones you can see, and says nothing about
  the rest.

## Meshblocks are roll-up-only

`meshblock` is the finest level the API holds. A meshblock value is **never returned
directly, to anyone, at any access level** — asking for one through any value, profile,
trend, ranking, distribution, correlation or comparison endpoint is refused as out of
scope.

Meshblocks can still feed an aggregate: [custom regions and area-group
roll-ups](../reference/rollup.md) are computed from them. That needs your grants to cover
meshblocks for the indicator, which the public baseline does not, so a request with no key
(or with a key whose access doesn't reach meshblocks) is refused with `401`/`403`.

To keep an aggregate from amounting to a single meshblock's value, a roll-up that draws on
meshblocks is refused with `422 INSUFFICIENT_DATA` unless its region contains enough of them —
see [Rollup → minimum size](../reference/rollup.md#minimum-size).

## Computed resources and scope

Comparisons, correlations and custom regions are persisted resources with their own URL
(`POST` creates, `GET /v1/.../{id}` retrieves). Scope is applied to **you, the caller**:

- The `result` returned inline with the `201` is computed against *your* scope.
- Every later `GET` re-resolves against the scope of **whoever makes that request**,
  intersected with the resource's stored inputs. A resource you created is not visible to
  you beyond what your scope allows at the time, and someone else with wider scope sees
  more.
- If none of a resource's stored inputs are visible to the caller, the `GET` is a
  `401`/`403` even though the resource exists.
- A comparison stores the Areas it resolved (by AUO Area ID), not the codes you sent, so a
  saved comparison keeps comparing the same Areas whichever ABS edition a code belonged to.
