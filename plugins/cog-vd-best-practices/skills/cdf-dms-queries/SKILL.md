---
name: cognite-dms-queries
description: Search-first, production-safe query patterns for Cognite DMS (Data Modeling Service), including pagination, traversal, retries, and anti-pattern avoidance.
---

# Cognite DMS Queries Skill

Use this skill when creating or reviewing queries against CDF DMS.

## Primary Goal

Produce queries that are:

1. Correct.
2. Efficient at scale.
3. Easy to reason about and maintain.

This skill is opinionated and **search-first**.

## Core Principles

1. **Search first, traverse second**
   - Use `instances.search(...)` to find/rank anchors.
   - Then hydrate related data with `Query` traversal or relation-aware filters.

2. **Constrain early**
   - Always scope with `space` and/or hard filters (`sourceContext`, subtree `Prefix`, etc.) whenever possible.
   - Avoid broad cross-space scans unless explicitly needed for discovery.

3. **Paginate explicitly**
   - Use cursor pagination (`/sync` style) or chunked iterators.
   - Prefer fixed page sizes (for example 500-1000) over large one-shot reads.

4. **Fetch only what you need**
   - In `select`, request explicit property lists.
   - Avoid wildcard projection (`"*"`).

5. **Operational resilience**
   - Retry transient failures (408/425/429/5xx) with bounded exponential backoff + jitter.
   - Keep query payloads and per-page limits moderate to reduce timeout risk.

## Recommended Query Flow

Follow this order by default:

1. Search basics (name-only scoring).
2. Constrained search (`query` + hard filters).
3. Top-K hydrate pattern.
4. Fallback strategy (strict -> broad).
5. Prefix-based subtree retrieval.
6. Graph traversals via `Query`.
7. Cursor pagination for full retrieval.

## Anti-patterns (do not generate unless explicitly requested)

1. `limit=-1` one-shot reads for large datasets.
2. `properties=["*"]` in production retrieval paths.
3. Broad `instances.list(...)` without `space` in production paths.
4. N+1 loops (querying related entities one by one in Python).
5. Client-side filtering when server-side filters can do the job.
6. Missing retry handling for transient API failures.
7. Mixed verbose/raw dumps (`print(res)` / full object dumps) for large responses.
8. Raw HTTP payload posts when SDK APIs provide equivalent capability and retries.
9. Filtering on `lastUpdatedTime` or `createdTime` on `/list` or `/query` for anything other than very small datasets — these base properties are indexed but the docs warn against filtering on them. Use `/sync` for change-tracking instead.
10. Sorting on non-cursorable properties for result sets larger than a few thousand instances — the default sort on `/list` is the internal ID, so any custom sort *must* be backed by a cursorable index or the query is likely to time out.

## Query performance — what actually costs

Understanding the join and sort model prevents most performance surprises.

- **Every mapped container = one join.** A view that maps 5 containers costs 5 joins per row selected. Keep views narrow, or accept the cost knowingly.
- **Every `nested` filter adds two joins.** Nesting is expressive but expensive at scale.
- **`hasData` across N containers in a multi-container view can materialize N joins** unless `requires` constraints let the planner short-circuit the check. This is why `requires` on multi-container views is not optional (see `cdf-data-model-indexes.md` and `cdf-data-model-structure.md`).
- **The `/list` endpoint's default sort is the internal ID.** Only `space` filters and `hasData` filters perform well against the default sort. To use `/list` with any other filter at scale, add a cursorable index whose property order matches the query's sort.
- **Cursoring by `space` is always performant** — `space` has a cursorable btree index by default. Use it for full-project backfills.

## Debug notices — the primary evidence tool

`/query` and `/sync` support a `debug` block that surfaces why a query is (or isn't) fast. Enable it whenever a query is slow or when you're validating an index design.

```python
res = client.data_modeling.instances.query(
    query=q,
    # SDK-equivalent debug options; see /cdf/dm/dm_guides/dm_debug_query_performance
    # for the raw request body shape.
)
# Or, via raw request body:
# {"debug": {"profile": true, "emitResults": false, "timeout": 30000}}
```

Notices carry a grade **A–E** (A = best practice, E = critical). Set `profile: true` (and `emitResults: false` for large queries) to see the full spectrum. The most common notices to expect:

| Notice | What it means | Fix |
|---|---|---|
| `sortNotBackedByIndex` | Query sorts on a property with no cursorable index; DMS does an in-memory sort | Add a cursorable btree matching the query's sort order (property order matters) |
| `unindexedThrough` | A `through` traversal targets a non-indexed direct relation | Add a btree index on the target property |
| `significantPostFiltering` | Filters are too late in the pipeline; too many intermediate rows | Move selective filters earlier — put space/type/`hasData` on the first `with` step |
| `significantHasDataFiltering` | `hasData` over a multi-container view forces per-container joins | Add `requires` constraints so the planner can shortcut them |

Advanced (alpha, format not stable — don't build production tooling on top):

- `includePlan: true` — return the underlying PostgreSQL execution plan.
- `translatedQuery: true` — return the translated internal query representation.

## `/sync` — modes and cursor lifetime

`/sync` is the correct tool for keeping a downstream system up to date with instance changes. Filter/sort trade-offs differ from `/query`:

| Mode | When to use |
|---|---|
| `onePhase` (default) | No filter or single `space` filter |
| `twoPhase` | Filter uses `hasData` or a cursorable index; splits backfill from live changes and lets an index accelerate the backfill (`backfillSort` matches the desired index) |
| `noBackfill` | Skip the initial backfill; only yield changes since the sync started |

Additional `/sync` constraints:

- **Sorting is not supported** while syncing — any sort would conflict with the ordering of new changes.
- **Cursor lifetime is 3 days.** After 3 days, soft-deleted instances are hard-deleted, so an expired cursor risks missing deletes.
- Set `allowExpiredCursorsAndAcceptMissedDeletes: true` if you explicitly accept the risk; otherwise, restart the sync when a cursor expires.

## Latency variability — expect long tails

DMS `/list` and `/query` latencies are not constant. A typical distribution is p50 ~200 ms / p90 ~1.5 s / **p99 ~4.5 s** — outliers are normal, not a service issue.

Design implications:

- **Don't require every call to complete within a strict time budget.** Interactive UX should show progress or partial results; background jobs should use long timeouts + retries.
- **Long tail latencies spike after schema-cache reloads.** Cleaning up **unused view versions** reduces the cost of full-project schema reloads, and therefore reduces tail latency. Treat orphan-view cleanup as a query-performance activity, not just a governance one.
- **Reduce payload with `select` selectors.** Only request the properties the caller actually needs — large payloads inflate serialization + network cost.
- CDF provides *availability* guarantees, not per-request *latency* guarantees.

## `hasData` — precise semantics

`hasData` filters accept a list of container refs, view refs, or both, ANDed together:

- **Container ref** matches when the instance has all *required* properties populated for that container.
- **View ref** without an explicit view filter matches when the instance has data in all the view's mapped containers (AND).
- **View ref** with an explicit view filter uses that filter *instead of* the implicit `hasData`.

This matters when you're chasing a `significantHasDataFiltering` notice — if a container has few required properties, `hasData` on the container may be cheaper than `hasData` on a wider view.

## Output and Style Requirements

When providing query examples:

- Include a short purpose statement.
- Prefer compact output helpers:
  - print count
  - print first N rows (`N=10`) with `externalId`, `name`, `description` (plus optional extra fields)
  - print overflow indicator (`... and N more`)
- Use discovery-first where practical (derive real token/value before applying strict filters).

## Reference

Use the canonical examples in:

- `references/cdf-dms-queries.md`

