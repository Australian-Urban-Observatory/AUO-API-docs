# Variants & percentile basis

Almost every endpoint takes a `variant` parameter (default `r`). A variant is *which
representation* of an indicator's value you want for a given area and year — not a different
indicator, and not a different year.

| Variant | Meaning | Peer group |
|---|---|---|
| `r` | Raw measured value | — |
| `p` | Percentile | National (every area in the country) |
| `pl` | Percentile within LGA | The area's own LGA |
| `pc` | Percentile within city | The area's own study region |

Every indicator is addressed with the same four variants (`variants` on
[indicator metadata](../reference/metadata.md) always lists all of them). A variant that
isn't one of these four returns `422 VARIANT_NOT_SUPPORTED`. A valid variant with nothing
recorded for the indicator, area and year simply comes back `null` — see
[`null` values](#null-values).

!!! warning "The same question has different answers depending on variant"
    "Which suburb is most liveable?" is ambiguous until you pick a basis: the raw score, the
    national percentile, the percentile within its own LGA, or the percentile within its own
    city. Switching basis can change the winner. Always be explicit in your own UI/output
    about which variant you used.

## `pl` and `pc` come from the data, not from the area's parents

`pl` and `pc` are percentiles the data source computed against its own peer groups. The API
serves one wherever the source recorded it — including 2018, whose peer groups the source
never recorded. An area is **not** refused a `pl` or `pc` just because the API holds no LGA or
study-region parent for it, and a variant with nothing recorded is a `null` value, never
`VARIANT_NOT_SUPPORTED`.

This is why `pl`/`pc` for a 2018 suburb or SA1 comes back with a value even though 2018 areas
have no [assigned parents](areas.md#assigned-parents): the percentile was computed upstream, and
the API reports it as stored.

## Custom regions only ever support `r`

A custom-drawn region (see [Rollup](../reference/rollup.md)) has no defined peer group of its
own, so percentile variants have no meaning there. Requesting anything other than `r` against
a custom region returns `422` with `CUSTOM_REGION_VARIANT_NOT_SUPPORTED` (a distinct code from
the generic `VARIANT_NOT_SUPPORTED` used everywhere else, specific to custom regions).

## `null` values

A `null` value means **no value is recorded** for that exact indicator/area/year/variant
combination — it does not mean zero, and it does not mean an error. Common reasons:

- The indicator wasn't computed for that year's release
- The area is too small/new for that indicator's methodology
- (Rare) the area or indicator was added after that release

Note this is different from an area that has **no feature at all** in a year — there, a
[trend](../reference/trend.md) leaves the year out and says so with `AREA_NOT_IN_YEAR`
instead of returning a `null` point.

If you need to know *why* a specific indicator has broad `null` coverage, ask the AUO team.
