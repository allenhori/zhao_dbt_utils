# zhao_utils

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

A small, growing collection of dbt macro utilities from the [`zhao`](https://github.com/allenhori/zhao-cli)
family. The first (and so far only) one is `wref()` ("windowed ref") — a `ref()`-style call to an
upstream model that actually uses the widened window a
[`zhao-dbt-plan`](https://github.com/allenhori/zhao-dbt-plan) plan computes, instead of dbt's own
default single-batch filter silently under-reading it. Separately licensed (Apache-2.0) from
`zhao-dbt-plan` itself (AGPLv3) — see ["Why a separate license"](#why-a-separate-license) below.
More utilities may land here over time as the need comes up — hence its own repo, not a
subdirectory of any one tool.

## The problem this solves

`zhao-dbt-plan` computes the *correct* cascading window for every downstream model — but on its
own, it can't make dbt's compiled SQL actually *use* that window. dbt's microbatch `ref()`
filtering can't be overridden by a project macro (it's resolved outside normal macro dispatch, to
build the dependency graph statically before Jinja even renders — confirmed against dbt-core's
actual behavior, not assumed), so a plain `{{ ref('model_a') }}` to an upstream still gets dbt's
own narrow, single-batch window — silently. No error, no warning, just a rolling-window model
quietly computing on too little data.

`wref()` is that missing piece.

## Is this for you? (completely optional)

If you've already solved this yourself — hand-writing `{{ ref('model_a').render() }}` plus your
own custom `WHERE` clause, the documented dbt pattern for rolling-window microbatch models — you
don't need this package at all. It changes nothing about how `zhao-dbt-plan` itself works, and
your existing models keep working exactly as they do today. This exists for whoever's starting
that pattern fresh, or wants to stop hardcoding day-counts by hand and read them from `meta.zhao`
instead. Use whichever of the three macros below fits how you already write SQL — or none of
them.

## Install

```yaml
# packages.yml
packages:
  - git: "https://github.com/allenhori/zhao_dbt_utils"
    revision: v0.1.0  # pin to a tag
```

Then `dbt deps`. (Not yet on dbt Hub — see [Publishing](#publishing) below.)

**Call these with the package namespace prefix** — `{{ zhao_utils.wref(...) }}`, not bare
`{{ wref(...) }}`. Confirmed against a real dbt-core project that bare calls to an installed
package's macros don't resolve without extra setup; namespaced calls do, always, with zero setup.
This matches how most dbt packages document themselves (e.g. `{{ dbt_utils.star(...) }}`). If you
want bare calls anyway, see ["Bare calls, if you want them"](#bare-calls-if-you-want-them) below —
there's a real, tested way to get that, just not for free.

## Naming: `expand_back`/`expand_forward`, not `lookback`/`lookahead`

This package's own arguments are deliberately **not** called `lookback`/`lookahead`, even though
that's what `meta.zhao` itself calls them (unchanged, see below) — to avoid confusion with dbt's
own unrelated native `lookback` model config, which means something completely different
("reprocess N past batches for late-arriving data," not "widen this read"). A bare
`lookback=N` argument sitting on the same model as dbt's own `config(lookback=N, ...)` would be a
real, easy mix-up. `expand_back`/`expand_forward` map directly to `meta.zhao`'s `lookback`/
`lookahead` keys internally — `meta.zhao` itself is unchanged and not renamed, since it's already
namespaced (`meta.zhao.lookback`, not bare `lookback`) and already established/documented within
`zhao-dbt-plan` itself.

## Usage

### `wref(upstream_name)` — "windowed ref," a drop-in `ref()` replacement

```sql
select * from {{ zhao_utils.wref('mb_daily') }} mb_daily
```

If the **current** model (the one calling `wref()`, not `mb_daily`) has a `meta.zhao` block, this
automatically applies its widened window. If it doesn't, this behaves *exactly* like plain
`{{ ref('mb_daily') }}` — no meta.zhao, no behavior change, completely safe to use everywhere as
a habit. Returns a derived-table subquery, so alias it at the call site like any other subquery.

**Important**: `meta.zhao` lives on the model doing the *reading* (the downstream/current model),
not on the upstream being read — e.g. if `mb_rolling_7d` calls `wref('mb_daily')`, the
`meta.zhao` block belongs on `mb_rolling_7d` itself, describing how far back/forward *it* needs
to read from its upstreams. Use `lookback_overrides`/`lookahead_overrides` (keyed by the
upstream's bare name) for per-upstream overrides of the model's own default. This matches
`zhao-dbt-plan`'s own planner semantics exactly.

### `zhao_window_start(upstream_name)` / `zhao_window_end(upstream_name)` — boundary helpers

If you already hand-write a custom `WHERE` clause using dbt's own `.render()` opt-out (the
documented pattern for rolling-window microbatch models), use these instead of hardcoding the
day-count:

```sql
select * from {{ ref('mb_daily').render() }}
where event_date >= {{ zhao_utils.zhao_window_start('mb_daily') }}
  and event_date <  {{ zhao_utils.zhao_window_end('mb_daily') }}
```

### Optional explicit arguments — visible-in-SQL numbers, still one source of truth

`expand_back`/`expand_forward` are **always optional**, on all three macros, in every combination
below. Pick whichever row matches how you want to write a given model — there's no wrong answer,
and you can mix approaches freely between different models in the same project.

```sql
-- explicit, visible at the call site:
select * from {{ zhao_utils.wref('mb_daily', expand_back=3, expand_forward=4) }} mb_daily

-- implicit, reads meta.zhao silently:
select * from {{ zhao_utils.wref('mb_daily') }} mb_daily
```

Every one of the five combinations below applies identically to `wref()`,
`zhao_window_start()`, and `zhao_window_end()` — they all share the same underlying resolution
logic, just returning a full filtered subquery vs. a single boundary expression.

| # | `expand_back`/`expand_forward` given? | `meta.zhao` on the current model? | What happens |
|---|---|---|---|
| 1 | No | No | Plain dbt default behavior — `wref()` returns exactly what `ref()` would; the boundary helpers return the batch's own unmodified `event_time_start`/`event_time_end`. No widening, no error, no warning. This is what you get by using these macros as a habit on models that don't need windowing at all. |
| 2 | No | Yes | Reads the widened window straight from `meta.zhao` (its `lookback`/`lookahead`, or the per-upstream override in `lookback_overrides`/`lookahead_overrides` if one exists for the model you're reading). Nothing visible in the SQL itself — the numbers live only in the config. This is the fully declarative, "configure once" way to use the package. |
| 3 | Yes | Yes, and it matches | Uses the value you gave. Since it matches `meta.zhao`, this is purely a legibility choice — the window is now visible right at the call site, and it's checked against the config on every compile, so it can never silently drift out of sync. |
| 4 | Yes | Yes, but it *doesn't* match | **Compile error.** The SQL and `meta.zhao` have diverged — one of them is wrong, and this fails loudly rather than silently using either value. Fix by updating one to match the other. |
| 5 | Yes | No | **Compiles and runs fine**, using the value you gave. `zhao-dbt-plan`'s planner has nothing to read for this model though, so **warns** (not an error) that its plan will be inaccurate here. This is the legitimate "I don't use the planner, I just want a convenient windowed read" case — fully supported, just flagged so you know what you're not getting. |

Rows 1 and 2 need nothing at the call site beyond the bare `wref('mb_daily')`/
`zhao_window_start('mb_daily')` call itself — whether you get widening or not depends entirely
on whether `meta.zhao` exists, set once in the model's own `config(...)` block. Rows 3-5 are only
reachable by deliberately passing `expand_back`/`expand_forward` yourself.

## Bare calls, if you want them

You can get `{{ wref(...) }}` working with no `zhao_utils.` prefix — but it takes one small file
in *your own* project, not a `dbt_project.yml` change. Tested and confirmed working:

```sql
-- your own project's macros/wref.sql
{% macro wref(upstream_name, expand_back=none, expand_forward=none) %}
  {{ return(zhao_utils.wref(upstream_name, expand_back, expand_forward)) }}
{% endmacro %}
```

That's it — a plain macro in your own project, calling straight through to the namespaced
package macro. Root-project macros are always resolvable bare, so this works with zero other
configuration. Two things worth knowing:

- **`adapter.dispatch()`/a `dispatch:` block in `dbt_project.yml` does *not* achieve this on its
  own** — tested directly, and it doesn't. `dispatch` controls which *implementation* an
  already-namespaced call resolves to (e.g. letting you override a package macro's behavior in
  your own project); it was never a mechanism for eliminating the namespace prefix at the call
  site itself.
- Add the same one-line wrapper for `zhao_window_start`/`zhao_window_end` if you want those bare
  too, following the identical pattern.

## v1 scope: `day`/`week` only

`meta.zhao`'s `lookback_unit`/`lookahead_unit: month` or `year` raise a clear compile error rather
than silently producing an approximate (and possibly wrong) result. Calendar month/year
arithmetic is warehouse-dialect-dependent, real scope, and deliberately deferred rather than
half-implemented.

## Why a separate license (and a separate repo)

`zhao-dbt-plan` itself is AGPLv3. This package is Apache-2.0, on purpose. AGPL's copyleft trigger
is about running/modifying *the program* — but this package's macros get copied directly into
*your* dbt project and compiled alongside your own models, which is a structurally different
situation from running `zhao-dbt-plan` as a separate tool. Apache-2.0 avoids any risk of this
package's license reaching into your own project just because you installed a few macros. Living
in its own repo (rather than a subdirectory of `zhao-dbt-plan`) keeps that license boundary clean
and obvious, and leaves room to grow into a real general-purpose utility collection later without
it being tied to one specific tool's release cycle.

## Tested against

Empirically verified, not assumed:
- **dbt-core 1.10.22 + DuckDB**: full real `dbt build` run, both the drop-in `wref()` path and
  the explicit-args-without-meta.zhao warning path, compiled SQL manually inspected and confirmed
  correct (`where order_date >= (batch_start - 3 days) and order_date < (batch_end + 4 days)`
  for a `lookback: 3, lookahead: 4` config). Also confirmed: bare-call resolution genuinely fails
  without a wrapper macro, and genuinely works with one (see above).
- **dbt Fusion 2.0.0-preview.203 + a real Databricks workspace**: compile-time verified —
  correct derived-table structure, correct `expand_back`/`expand_forward` direction, and correct
  per-adapter SQL dialect dispatch (`dbt.dateadd` compiled to Databricks-native
  `timestampadd(day, ...)`, vs. DuckDB's `+ interval` syntax under dbt-core — confirming
  cross-adapter portability). A full real *run* under Fusion against Databricks was blocked by an
  unrelated, pre-existing dbt Fusion/Databricks microbatch materialization bug (reproduced with
  this package completely uninstalled, on a model that never calls any of these macros — not
  something this package caused or can fix).

## Publishing

Not yet submitted to [dbt Hub](https://hub.getdbt.com) — planned as a follow-up once this has
had more real-world use beyond the initial testing.
