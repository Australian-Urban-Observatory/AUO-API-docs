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

Not every indicator defines every variant — check `variants` on
[indicator metadata](../reference/metadata.md) before requesting one. Requesting an
undefined variant returns `422 VARIANT_NOT_SUPPORTED`.

!!! warning "The same question has different answers depending on variant"
    "Which suburb is most liveable?" is ambiguous until you pick a basis: the raw score, the
    national percentile, the percentile within its own LGA, or the percentile within its own
    city. Switching basis can change the winner. Always be explicit in your own UI/output
    about which variant you used.

## Why `pl`/`pc` can be unavailable for a specific area

`pl` needs the area to have an LGA parent; `pc` needs a study region parent. An area with no
such parent (e.g. a state-level or study-region-level area itself has no *containing* LGA)
returns `422 VARIANT_NOT_SUPPORTED` for that variant — this is a property of the *area*, not
just the indicator.

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

If you need to know *why* a specific indicator has broad `null` coverage, check
[Admin → coverage gaps](../reference/admin.md#coverage-gaps) (requires an admin key), or ask
the AUO team.
