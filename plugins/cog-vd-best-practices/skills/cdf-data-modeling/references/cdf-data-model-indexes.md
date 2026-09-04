# CDF Data Model Indexing Best Practices

These rules apply to **`usedFor: node`** containers only. **`usedFor: record`** containers must **not** define `indexes` (records are not tuned as graph nodes for reverse relations or btree-backed view queries).

## Index Types

CDF supports two index types on `usedFor: node` containers:

- **`btree`** — for scalar and specific list types. Supports equality, range, and (when `cursorable: true`) ordered pagination. Optional `bySpace: true` prefixes the index with `node.space`, which is useful when queries are usually scoped to a single space.
- **`inverted`** — for **list-type properties** (e.g. `text[]`, `int64[]`, `direct[]`). Enables `containsAny` / `containsAll` filtering across the elements of the list. **Inverted indexes cannot be `cursorable`**.

Pick `btree` for scalar filters, ranges, sorts, and direct-relation traversal. Pick `inverted` when the property is a list and consumers filter on individual elements.

## Index Limits
- Max **10 indexes per container** (`usedFor: node`). Use slots strategically — don't index everything.
- Never modify indexes or constraints on CDM types (`cdf_cdm` space) — they are immutable.

## Default-indexed base properties

Every node and edge already has btree indexes on the following base properties — you do not need (and cannot) add them yourself:

| Property | Cursorable |
|---|---|
| `space` | Yes |
| `externalId` | Yes |
| `type` | No |
| `startNode` (edges) | No |
| `endNode` (edges) | No |

Implications:

- Cursoring through **all instances scoped by `space`** is performant out of the box — use it for full backfills.
- Filters on `type`, `startNode`, `endNode` are fast but **cannot cursor** — pair them with a cursorable custom index if you need to page results.
- Filters on **`lastUpdatedTime`** and **`createdTime`** are **not** performant except on very small datasets. The Cognite docs explicitly warn against using them in DMS `/list` and `/query` filters. Do not design workflows that rely on `lastUpdatedTime` filtering — use `/sync` (which subscribes to changes) instead. See `cdf-dms-queries` for the query-side pattern.

> Transformations note: The `is_new()` function inside CDF Transformations SQL uses `lastUpdatedTime` as the cursor argument. That is a *different* mechanism — it is not a DMS filter, so the warning above does not apply there. See the `cognite-transformation` skill.

## What to Index

### Filterable Properties
Index properties used in filters: `name`, `number`, discipline codes, status fields.

### Range Queries
Index timestamp and numerical properties used in range queries (e.g., `planned_start_time`, `start_time`, `end_time`, `test_date`).

### Composite Indexes
Create composite indexes for common query patterns — especially **status + timestamp** combinations. This avoids scanning all records of one status to find a time range.

### Relationship Properties
Index direct relation properties used for bidirectional navigation (e.g., `tag`, `asset`, `workOrder`). This ensures fast traversal in both directions.

### Direct Relation Index Priority (MUST/SHOULD)
- **MUST** index any direct relation used as `through.identifier` in a reverse relation.
- **MUST** index direct relations used in high-traffic filters, joins, or traversal paths in applications.
- **SHOULD** index other direct relations when they are expected to be queryable, even if currently low traffic.
- **EXCEPTION**: if a direct list relation has `maxListSize > 300`, btree indexing is not supported. Document the exception in the container YAML (comment near the property or indexes) and in module docs/README, and provide an alternative traversal strategy (for example, reverse from another indexed relation, a denormalized read view, or a companion scalar relation for filtering).

### Secondary Filter Properties
Index text properties commonly used as sidebar/facet filters: `work_type`, `equipment_class`, `material_class`.

### Baseline Index Hygiene for Core Entities
For core business entities (for example wells, work orders, equipment), define a minimal index baseline:
- one stable business identifier for lookup (for example `wellId`, `sourceId`, `tagNumber`)
- one or more status/type filter fields
- one cursorable date/timestamp field used for incremental reads/pagination

## Btree Index Type Restrictions

### List (array) properties
- Btree indexes support these list types only: `int32[]`, `int64[]`, `timestamp[]`, `direct[]`.
- **`text[]` cannot have a btree index** — use an `inverted` index instead.
- **List properties need `maxListSize` set** to be included in a btree, and the cap depends on the type:
  - `int32[]`: `maxListSize <= 600`
  - `int64[]` / `direct[]`: `maxListSize <= 300`
  - Other list types: `maxListSize <= 2000` overall, but only the types above are btree-eligible
- If `maxListSize` is not set or exceeds these caps, CDF rejects the btree index. Either lower `maxListSize`, remove the index, or use an `inverted` index (for `text[]`) — and document the trade-off.

### Text properties in btree indexes
- **`maxTextSize` <= 2400 bytes (UTF-8)** for a text property to be included in a btree index.
- **Combined limit across a composite btree**: the total size of all indexed properties on a single btree must not exceed 2400 bytes.
- Text properties with values larger than 2400 bytes cannot be btree-indexed — split the property, cap `maxTextSize`, or index a hashed/derived column instead.

### Cursorable
`cursorable: true` is only supported on **scalar** (non-list) types: `text`, `boolean`, `int32`, `int64`, `float32`, `float64`, `date`, `timestamp`, `direct`. Always set `cursorable: false` for list properties, and remember that inverted indexes cannot be cursorable at all.

## Large Read Strategy (Explicit Recommendation)
- Avoid unbounded reads (`limit: -1`) for operational queries; they are expensive at scale.
- For large datasets, use cursor pagination against a scalar property backed by a btree index with `cursorable: true`.
- Recommended pattern: fetch in stable batches (for example `limit: 1000`) and continue via cursor until complete.
- Choose a cursor field that supports deterministic progression (typically `lastUpdatedTime`, `createdTime`, or another monotonic scalar used in your query path).
- Do not use list properties as cursor fields; list properties must remain `cursorable: false`.

## What NOT to Index
- Properties already covered by unique constraints (e.g., `tag_number`) — these function as indexes.
- Properties never used in filters or sorts.
- CDM container properties — those are managed by Cognite.

## Direct Relations (Connections)

### End Node Types
Always specify the target view (`source`) for direct relation properties in views. This lets the system optimize storage and indexing for that relationship.

> Note: For direct relation properties sourced from CDM/IDM containers (`cdf_cdm` / `cdf_idm`), you cannot add or modify indexes in those managed containers. Treat index tuning there as informational and optimize through your own containers/views instead.

```yaml
# Good — explicit target view
assets:
  container:
    space: "{{space}}"
    externalId: MyContainer
    type: container
  containerPropertyIdentifier: tag
  source:
    space: "{{space}}"
    externalId: Tag
    version: "{{dm_version}}"
    type: view
```

### Reverse Relations
Every `multi_reverse_direct_relation` or `single_reverse_direct_relation` in a view relies on a `through.identifier` property in the target container. That container property **must** have a btree index — otherwise CDF warns: *"has a reverse direct relation that cannot be efficiently traversed through queries, since it points to container property that is unindexed."*

When adding a reverse relation to a view, always verify:
1. The `through.identifier` property exists in the target container
2. That property has a btree index in the target container's `indexes:` section

```yaml
# View defines reverse relation
workOrderOperations:
  through:
    source: { space: '{{space}}', externalId: WorkOrderOperation, ... }
    identifier: functionalLocation   # ← this property must be indexed

# Target container MUST have matching index
indexes:
  functionalLocation:
    indexType: btree
    properties:
      - functionalLocation
    cursorable: false
```

## Requires Constraints

`requires` constraints declare a logical dependency between containers and are **mandatory for query performance** on any view that spans multiple containers. Without them, the query planner cannot prune the joins triggered by a `hasData` filter across the view's mapped containers, and query performance is typically unacceptable at scale.

The Cognite docs' `significantHasDataFiltering` debug notice is emitted specifically when a `hasData` filter on a multi-container view has no `requires` chain to shortcut it. See `cdf-dms-queries` for details on debug notices.

```yaml
constraints:
  assetPresent:
    constraintType: requires
    require:
      space: cdf_cdm
      externalId: CogniteAsset
      type: container
```

**Do not** add unrelated `requires` constraints just to silence warnings — they are ingest-time dependencies and will make writes fail if the required container is absent. See `cdf-data-model-structure.md` → *Query Optimization via `requires`* for the semantic-validity rule.

## Denormalization for Search
Prefer flattened, denormalized access views as the default for read/search-heavy use cases. Keep semantics standardized through CDM/IDM, then expose commonly queried properties together in one view to avoid multi-hop joins and make data easier to consume. Only fall back to highly normalized multi-view query paths for niche or governance-constrained scenarios.

## Common Index Patterns by Entity

| Entity | Properties to Index |
|--------|-------------------|
| WorkOrder | `work_status`, `planned_start_time`, `actual_start_time` |
| Equipment | `equipment_name`, `equipment_type`, `serial_number` |
| Site | `site_name` |
| Batch | `batch_state`, `start_time`, `end_time` |
| QualityResult | `status`, `test_date`, batch relation |

## Deployment
Always define indexes in `containers/*.yaml` files and deploy via Cognite Toolkit to keep dev/test/prod consistent.

## Composite Indexes — Order Matters

Composite indexes (multiple properties on one index) are effective when consumers filter or sort on the properties together. **The order of properties inside the index is significant**: an index on `(site, name)` accelerates filters that lead with `site` (and optionally add `name`), but not filters that lead with `name` alone. If both query patterns exist, define two separate composite indexes.

- Composite indexes can only be built from properties in the **same container**.
- Do not oversize: a small number of well-chosen composite indexes beats many broad ones — each additional index slows every ingest.
- For a composite btree, the combined size limit is still 2400 bytes across all indexed properties (see *Btree Index Type Restrictions*).

## Debug notices — closing the loop from queries to indexes

The Cognite docs expose a first-class debugging tool for `/query` and `/sync` — enable with `debug: {}` on the request. The notices most relevant to indexing are:

- **`sortNotBackedByIndex`** — the query's sort has no cursorable index; add one whose property order matches the sort.
- **`unindexedThrough`** — a `through` traversal targets a non-indexed direct-relation property; add a btree index on that property.
- **`significantHasDataFiltering`** — a `hasData` filter over a multi-container view forces per-container joins; add `requires` constraints so the planner can shortcut them.
- **`significantPostFiltering`** — filters are too late in the query pipeline; move selective filters earlier (this is a query-shape problem, not an index one).

For handling these on the query side, see `cdf-dms-queries`.