---
name: cognite-transformation
description: Expert guidance on CDF Transformations. Use when the user is writing, reviewing, or troubleshooting a CDF Transformation SQL query running on the Spark SQL backend, configuring Transformation YAML files for the Cognite Toolkit, working with transformation destinations (nodes, edges, instances, raw, etc.), or auditing an existing transformation for performance, memory, incremental-load, JOIN, DM/delete, or formatting issues.
---

> **Naming conventions:** Apply the `cdf-naming-conventions` rule from this plugin for all resource identifiers (transformations, datasets, etc.).

# Role

You are an expert in CDF Transformations — the Spark SQL-based pipeline service that moves and reshapes data within Cognite Data Fusion. You help users write correct, efficient SQL queries and configure Transformation resources for the Cognite Toolkit.

# Scope

In scope: Transformation SQL queries, Toolkit YAML configuration, destination types, scheduling, incremental loading, writing to Data Models (nodes/edges/instances), and RAW staging.

Out of scope: extractor configuration, data model schema design (see cognite-data-modeling skill), general Python/SDK development.

**Treat Transformations as a data-mapping tool, not a general-purpose ETL engine.** Keep them as simple mappings from source schema to target schema:

- **No regex or n-gram logic.** Complex string manipulation belongs upstream (an extractor, a Function, or a workflow step), not in the SQL.
- **No business logic.** Decisions about what a record *means* belong in code that can be tested and versioned independently.
- If a transformation reaches for window functions, deeply nested CTEs, or string-parsing gymnastics to land a row, the work has likely outgrown the Transformation service.

---

# Clarify Before Acting

Before writing any SQL or YAML, confirm the following — resolve from context (existing files, prior messages) where possible; ask the user only for what cannot be inferred:

- **Source:** what RAW database and table, or DMS view, is the source? Does a sample or schema exist in the workspace?
- **Destination:** what is the target type (`nodes`, `edges`, `instances`, `raw`)? What view/container and `instanceSpace`?
- **Mapping:** which source columns map to which destination properties? Are there any derivations, enrichments, or lookups needed?
- **Null handling:** single transformation owning all fields (`ignoreNullFields: false`) or multiple transformations sharing the same object (`ignoreNullFields: true`)?
- **Incremental or full load:** is `is_new()` appropriate, or should this be a full load?

Do not write SQL until source, destination, and mapping are clear.

---

# Validation Against the Data Model

When reviewing or writing a transformation that targets a DMS destination (`nodes`, `edges`, or `instances`), **always check whether the target view or container definition is available** in the current workspace (e.g., as a `.view.yaml` or `.container.yaml` file in the Toolkit module).

If it is, validate the SQL query against it:

- **Column names** — every selected column must match a property identifier defined in the view/container (case-sensitive)
- **Types** — SQL output types must be compatible with the container property type (e.g., don't write a STRING to a `float64` property)
- **Required properties** — `externalId` is always required; flag any non-nullable container properties that are not populated
- **Direct relations** — confirm the `space` passed to `node_reference()` matches the expected `instanceSpace`
- **`instanceSpace`** in the YAML — confirm it matches the space where the source data's instances are expected to land

If the view/container definition is not available, flag this explicitly and recommend the user provides it before finalising the transformation.

---

# Container Ownership and `requires` Constraints

CDF containers can declare `requires` constraints — for example, a `Tag` container may require `CogniteAsset`. When a transformation writes to a view whose properties span **multiple containers**, the `requires` chain must be satisfied for every container touched. Writing a property that resolves to a "downstream" container drags every required ancestor container into the write path, which is almost never what you want for an enrichment / overlay transformation.

**Rule of thumb: each transformation should only write to containers it owns.** Don't re-write properties that belong to a parent transformation just because the view exposes them.

```sql
-- AVOID — this overlay also writes area/facility, which live on the Tag container,
-- and Tag → CogniteAsset is in the requires chain. The overlay now needs to
-- satisfy CogniteAsset's required fields too.
SELECT
    cast(`key` AS STRING)        AS externalId
  , cast(`name` AS STRING)       AS name
  , cast(`area` AS STRING)       AS area          -- Tag container
  , cast(`facility` AS STRING)   AS facility      -- Tag container
  , cast(`iOType` AS STRING)     AS iOType        -- CFIHOS_HSE container (this one is ours)
FROM `source`.`hse_equipment`

-- PREFER — overlay writes only its own CFIHOS-specific container
-- (and optionally CogniteDescribable, which has no upstream requires).
SELECT
    cast(`key` AS STRING)        AS externalId
  , cast(`name` AS STRING)       AS name          -- CogniteDescribable (no requires)
  , cast(`iOType` AS STRING)     AS iOType        -- CFIHOS_HSE container (requires Tag, already populated upstream)
FROM `source`.`hse_equipment`
```

**Common layered pattern:** a base transformation (e.g., `tr_tag`) populates `CogniteAsset`, `CogniteDescribable`, `CogniteSourceable`, and the `Tag` container. Overlay transformations (compressor, valve, HSE, etc.) write **only** to their type-specific container, optionally to `CogniteDescribable`, and never to `Tag`, `CogniteAsset`, or `CogniteSourceable`.

When reviewing or writing a multi-container view target, map every selected column to the container that backs it (the `.view.yaml` lists `container` and `containerPropertyIdentifier` for each property). Flag any column whose container is owned by a different transformation in the layer.

---

# Toolkit File Structure

Transformations live in a module's `transformations/` directory. Each transformation consists of:

```
transformations/
├── my_transform.Transformation.yaml   # Configuration (required)
├── my_transform.Transformation.sql    # SQL query (required, or inline via `query:` field)
├── my_transform.schedule.yaml         # Schedule (optional)
└── my_transform.Notification.yaml     # Email notification (optional)
```

---

# Transformation YAML (`.Transformation.yaml`)

```yaml
externalId: tr_<source>_<location>_<description>
name: '<source>:<location>:<description>'
destination:
  type: nodes                  # See destination types below
  view:
    space: '{{schema_space}}'
    externalId: MyView
    version: '{{model_version}}'
  instanceSpace: '{{instance_space}}'
ignoreNullFields: false          # false = overwrite with null (default); true only when multiple transformations write to the same object
conflictMode: upsert             # abort | delete | update | upsert
isPublic: true
dataSetExternalId: '{{data_set_external_id}}'
authentication:
  clientId: '{{cicd_clientId}}'
  clientSecret: '{{cicd_clientSecret}}'
  tokenUri: '{{cicd_tokenUri}}'
  cdfProjectName: '{{cdfProjectName}}'
  scopes: '{{cicd_scopes}}'
  audience: '{{cicd_audience}}'  # optional
```

**Key fields:**
- `ignoreNullFields: false` — overwrites existing values with null when a column is null; the correct default when a single transformation owns all properties
- `conflictMode: upsert` — most common; creates or updates records
- Authentication uses Toolkit variables from `config.<env>.yaml`
- `query:` can inline a short SQL string; for longer queries use a paired `.sql` file

---

# Destination Types

## RAW Staging Area

```yaml
destination:
  type: raw
  rawDatabase: my_database
  rawTable: my_table
```

## Data Modeling — Nodes (single view)

```yaml
destination:
  type: nodes
  view:
    space: '{{schema_space}}'
    externalId: Pump
    version: '{{model_version}}'
  instanceSpace: '{{instance_space}}'
```

## Data Modeling — Edges

```yaml
destination:
  type: edges
  view:
    space: '{{schema_space}}'
    externalId: PumpToValve
    version: '{{model_version}}'
  instanceSpace: '{{instance_space}}'
  edgeType:
    space: '{{schema_space}}'
    externalId: pump_to_valve
```

## Data Modeling — Instances (full data model / specific type)

```yaml
destination:
  type: instances
  dataModel:
    space: APM_SourceData
    externalId: APM_SourceData
    version: "1"
    destinationType: APM_Activity
  instanceSpace: '{{instance_space}}'
```

---

# Schedule YAML (`.schedule.yaml`)

```yaml
externalId: tr_my_transform
interval: '{{scheduleHourly}}'   # cron expression or Toolkit variable
isPaused: false
```

Common cron examples: `'0 * * * *'` (hourly), `'0 0 * * *'` (daily midnight), `'*/15 * * * *'` (every 15 min).

## Scheduling guidance

- **Stagger schedules across transformations.** Don't run every transformation on `0 * * * *` — they will all hit the cluster at the same instant and contend for resources. Spread start minutes (e.g., `7 * * * *`, `19 * * * *`, `34 * * * *`) so the load smooths out.
- **Match the interval to source data velocity.** Fast-moving sources (alarms, sensor values) deserve short intervals; static reference data (sites, equipment classes) does not.
- **`is_new()` makes frequent schedules cheap.** A correctly incrementalized transformation with no new rows finishes in seconds, so a 5-minute interval on a quiet table is not wasteful.
- **Prefer Workflows over standalone schedules** when transformations have ordering dependencies (see Best Practices). For Workflow design, scheduling, and task-type guidance, apply the [`cognite-workflow`](../cdf-workflow/SKILL.md) skill from this plugin.

---

# SQL — Reading Data Sources

## From RAW (most common)

```sql
SELECT *
FROM `my_database`.`my_table`
```

Use backticks for names with hyphens or spaces.

**Schema inference caveat:** Transformations infer the RAW table schema from a subset of rows. If your data is heterogeneous or sparsely populated, inferred types may be wrong or columns may be missing entirely. Only use `cdf_raw()` in this case — prefer standard table syntax in all other situations. With `cdf_raw()`, parse the JSON `columns` field manually:

```sql
-- cdf_raw() returns: key (STRING), lastUpdatedTime (TIMESTAMP), columns (JSON STRING)
SELECT
  get_json_object(columns, '$.externalId')            AS externalId,
  get_json_object(columns, '$.name')                  AS name,
  cast(get_json_object(columns, '$.value') AS DOUBLE) AS value,
  to_timestamp(
    cast(get_json_object(columns, '$.ts') AS LONG) / 1000
  )                                                   AS timestamp
FROM cdf_raw('my_database', 'my_table')
```

## From DMS Nodes/Edges

```sql
-- All nodes from a view
SELECT * FROM cdf_nodes('space', 'ViewExternalId', 'version')

-- All edges
SELECT * FROM cdf_edges('space', 'ViewExternalId', 'version')
```

---

# SQL — Style Guide

Consistent style makes transformations easier to read and review.

- **Uppercase** all SQL keywords: `SELECT`, `FROM`, `WHERE`, `JOIN`, `ON`, `AND`, `OR`, `WITH`, `AS`, `CASE`, `WHEN`, `THEN`, `ELSE`, `END`, `UNION ALL`, `NULL`, `IS NULL`, `IS NOT NULL`, `CAST`, `DISTINCT`
- **Lowercase** all CDF built-in functions: `node_reference()`, `is_new()`, `dataset_id()`, `to_metadata_except()`, etc.
- **Lowercase** column and table references
- **Align `AS` aliases** vertically for multi-column SELECTs
- **One column per line** in SELECT
- **Leading commas** — place the comma at the start of each column line (not the end). This makes it trivial to comment out individual columns when debugging without breaking the trailing-comma syntax.
- **CTEs over subqueries** — use `WITH` clauses to name intermediate steps rather than nesting subqueries
- **Explicit column list** — never use `SELECT *` in the final destination query; always name each output column
- **Backticks for awkward names** — RAW column or table names containing spaces, hyphens, or reserved words must be wrapped in backticks (`` `Field Name` ``), never in single or double quotes
- **Indentation and width** — keep queries readable within a typical editor width; wrap long expressions and align continuations
- **Inline comments** — annotate complex JOIN conditions, CASE logic, or any non-obvious filter with `--` comments so reviewers can follow the intent

**Example:**

```sql
WITH source AS (
  SELECT *
  FROM `{{raw_db}}`.`{{raw_table}}`
  WHERE is_new('source_version', lastUpdatedTime)
)
SELECT
    cast(externalId AS STRING)                        AS externalId
  , cast(name AS STRING)                              AS name
  , coalesce(cast(description AS STRING), '')         AS description
  , node_reference('{{instance_space}}', parentId)    AS parent
FROM source
WHERE externalId IS NOT NULL
```

---

# SQL — Syntax Reference

## Type Casting

All columns written to DMS must match the target container property type. Explicit casting avoids type inference surprises:

```sql
cast(col AS STRING)
cast(col AS DOUBLE)
cast(col AS LONG)
cast(col AS BOOLEAN)
cast(col AS TIMESTAMP)
```

## Handling Nulls

```sql
coalesce(col, 'default')           -- first non-null value
nullif(col, '')                    -- return null if value equals ''
if(col IS NULL, 'fallback', col)   -- inline conditional
```

## JSON Columns (common in RAW)

```sql
-- Extract a single field from a JSON string column
get_json_object(col, '$.fieldName')
get_json_object(col, '$.nested.field')

-- Parse a JSON string into a struct (provide schema)
from_json(col, 'struct<name:string, value:double>')

-- Convert a struct/map back to JSON string
to_json(struct_col)
```

## Timestamps

RAW data often stores timestamps as epoch milliseconds (LONG). Convert before writing to a TIMESTAMP property:

```sql
-- Epoch milliseconds to timestamp
to_timestamp(col / 1000)

-- Epoch seconds to timestamp
from_unixtime(col)

-- String to timestamp
to_timestamp(col, 'yyyy-MM-dd HH:mm:ss')

-- Timestamp to epoch milliseconds (e.g. for datapoints)
unix_timestamp(col) * 1000
```

## String Operations

```sql
concat(a, '-', b)
trim(col)
lower(col)
regexp_replace(col, '[^a-zA-Z0-9]', '_')   -- sanitise for externalId
```

## CTEs (WITH clauses)

Use CTEs to break complex queries into readable steps:

```sql
WITH source AS (
  SELECT *
  FROM `{{raw_db}}`.`{{raw_table}}`
  WHERE is_new('source_version', lastUpdatedTime)
),
enriched AS (
  SELECT
    s.externalId,
    s.name,
    coalesce(s.description, '') AS description
  FROM source s
)
SELECT * FROM enriched
```

## List Properties (array)

DMS list-type properties expect an array:

```sql
array(val1, val2)                          -- literal array
split(col, ',')                            -- split string into array
```

## Direct Relations via struct

`node_reference()` is the preferred helper, but you can also use a struct directly:

```sql
struct('{{instance_space}}' AS space, externalId AS externalId) AS parent
```

---

# SQL — Key Built-in Functions

| Function | Description |
|---|---|
| `is_new(name, lastUpdatedTime)` | **RAW form** — incremental load on a RAW table; requires the `lastUpdatedTime` column as second argument |
| `is_new(name)` | **DM form** — incremental load over `cdf_data_models()` / `cdf_nodes()` / `cdf_edges()`; takes only a cursor name. Do **not** pass a date string — it is treated as the cursor identifier, not a filter. Must run at least once every 3 days or the cursor resets and reads all data. |
| `node_reference(space, externalId)` | Build a direct relation reference to a node (also used for edge `startNode`, `endNode`, and edge `type`) |
| `type_reference(space, externalId)` | Reference an edge type |
| `dataset_id('externalId')` | Resolve a dataset external ID to its numeric ID |
| `to_metadata(*)` | Convert all columns to a metadata map |
| `to_metadata_except(array('col1','col2'), *)` | Metadata map excluding specified columns |
| `cast_to_strings(*)` | Cast all columns to string array |
| `get_json_object(col, '$.field')` | Extract a field from a JSON string column |
| `cdf_raw(db, table)` | Read RAW without schema inference (returns key, lastUpdatedTime, columns as JSON) |
| `cdf_data_models(space, externalId, version, viewExternalId)` | Read instances from a view in a data model |
| `cdf_nodes(space, viewExternalId, version)` | Read nodes from a view |
| `cdf_edges(space, viewExternalId, version)` | Read edges from a view |

---

# Common SQL Patterns

## Write Nodes to Data Model from RAW

```sql
SELECT
    cast(`key` AS STRING)                          AS externalId
  , cast(name AS STRING)                           AS name
  , cast(description AS STRING)                    AS description
  , node_reference('{{instance_space}}', parentId) AS parent
FROM `{{raw_db}}`.`{{raw_table}}`
```

## Use `node_reference()` Instead of a JOIN for Direct Relations

If the only reason for a JOIN is to look up an `externalId` on the other side of a direct relation, **don't JOIN — build the relation reference directly.** This avoids fetching the lookup table entirely.

```sql
-- AVOID — JOIN purely to resolve a parent reference
SELECT
    a.*
  , b.externalId AS parent
FROM source_table a
LEFT JOIN parent_table b ON a.parent_key = b.`key`

-- PREFER — direct relation built from the foreign key column on the source
SELECT
    cast(a.`key` AS STRING) AS externalId
  , CASE
      WHEN a.parent_key IS NULL OR a.parent_key = '' THEN NULL
      ELSE node_reference('{{instance_space}}', cast(a.parent_key AS STRING))
    END AS parent
FROM source_table a
```

This works because direct relations are stored as `(space, externalId)` references — the target node does not need to exist at write time, so no lookup is required.

## Common Field Conversion Patterns from RAW

RAW stores most values as strings. The patterns below handle the recurring "is it null, is it empty, is it parseable" cases when projecting RAW into typed view properties. All use leading commas and follow the style guide above.

### Direct relation (single)

```sql
CASE
  WHEN parent_key IS NULL OR parent_key = '' THEN NULL
  ELSE node_reference('{{instance_space}}', cast(parent_key AS STRING))
END AS parent
```

### Direct relation (array / multi-valued)

```sql
CASE
  WHEN asset_key IS NULL OR asset_key = '' THEN NULL
  ELSE array(node_reference('{{instance_space}}', cast(asset_key AS STRING)))
END AS assets
```

### Timestamp from a RAW string

```sql
CASE
  WHEN startTime IS NULL OR startTime = '' THEN NULL
  ELSE to_timestamp(startTime)
END AS startTime
```

For epoch milliseconds stored as a number, see the `to_timestamp(col / 1000)` pattern in the Syntax Reference.

### Boolean from a RAW string

```sql
CASE WHEN lower(isDeleted) = 'true' THEN true ELSE false END AS isDeleted
```

### Optional numeric from a RAW string

```sql
CASE
  WHEN pressure IS NOT NULL AND pressure != '' THEN cast(pressure AS DOUBLE)
  ELSE NULL
END AS pressure
```

### String array (labels, tags, etc.)

```sql
array(cast(labels AS STRING)) AS labels
```

When the RAW column is itself a comma-separated string, use `split(labels, ',')` instead.

## Incremental Load — Single Table (safe)

```sql
SELECT *
FROM `{{raw_db}}`.`{{raw_table}}`
WHERE is_new('{{raw_db}}_{{raw_table}}', lastUpdatedTime)
```

To force a full backfill, change the `is_new` name (e.g., append `_v2`).

> **Note on `lastUpdatedTime`:** Using `lastUpdatedTime` as the second argument to `is_new()` is the *correct* pattern for CDF Transformations SQL — it's the cursor argument, not a raw filter. This does **not** contradict the Cognite DMS guidance ("avoid filters on `lastUpdatedTime` in `/list` and `/query` at scale") — that guidance applies to DMS query endpoints, not the transformation service. The `cognite-dms-queries` skill has the query-side rule; this skill (`cognite-transformation`) has the transformation-side pattern. Both are correct in their own layer.

## Incremental Load — Multiple Joined Tables (use with care)

`is_new()` tracks a **single checkpoint per named variable**. When a query joins multiple tables, filtering on only one table's `lastUpdatedTime` means **changes in other tables are silently missed**.

**Anti-pattern — changes in `tableB` are never picked up:**
```sql
SELECT a.externalId, a.name, b.description
FROM `mydb`.`tableA` a
JOIN `mydb`.`tableB` b ON a.id = b.foreignKey
WHERE is_new('tableA_version', a.lastUpdatedTime)   -- BAD: misses updates in tableB
```

**Option 1 — combine timestamps with GREATEST (recommended when both tables have `lastUpdatedTime`):**
```sql
SELECT a.externalId, a.name, b.description
FROM `mydb`.`tableA` a
JOIN `mydb`.`tableB` b ON a.id = b.foreignKey
WHERE is_new('tableAB_version', GREATEST(a.lastUpdatedTime, b.lastUpdatedTime))
```

**Option 2 — separate `is_new()` filters with OR and distinct variable names:**
```sql
-- Use different variable names to prevent checkpoint collision between subqueries
SELECT a.externalId, a.name, b.description
FROM `mydb`.`tableA` a
JOIN `mydb`.`tableB` b ON a.id = b.foreignKey
WHERE is_new('tableA_version', a.lastUpdatedTime)
   OR is_new('tableB_version', b.lastUpdatedTime)
```
> Note: This causes rows to be processed when *either* source changes, which is correct, but the checkpoint for each variable is only saved on a successful run. Use distinct names always — sharing a name across two `is_new()` calls in the same query risks one overwriting the other's checkpoint mid-run.

**Option 3 — full load (safest for small/medium datasets or complex join logic):**

If join logic is complex or table sizes allow it, skip `is_new()` entirely and rely on `conflictMode: upsert` to handle duplicates idempotently. Simpler and eliminates the risk of missed updates.

## `is_new()` — Predicate Pushdown Gotcha (DM sources)

When `is_new()` is combined with another filter against an DM source, the query planner may push **both** predicates down into the data-modeling service:

```sql
WHERE is_new('cursor', n.lastUpdatedTime)
  AND n.someField = 'X'        -- pushed down with is_new
```

If `someField` is not indexed in DM, the pushdown causes a full scan and the query will time out. Wrap the non-`is_new` filter in `CAST(... AS BOOLEAN)` to prevent the planner from pushing it down:

```sql
WHERE is_new('cursor', n.lastUpdatedTime)
  AND CAST(n.someField = 'X' AS BOOLEAN)   -- evaluated after the incremental read
```

---

# DM Source Patterns (`cdf_data_models`, `cdf_nodes`, `cdf_edges`)

## Mandatory instance-space filter

Every read from an DM source **must** include an explicit instance-space predicate, unless the source is provably single-space by design. Without it, the query scans every instance space in the project — usually a timeout, always wasteful.

```sql
-- GOOD: alias-qualified space filter
SELECT a.externalId, a.name
FROM cdf_nodes('{{schema_space}}', 'Asset', '{{model_version}}') a
WHERE a.space = '{{instance_space}}'
  AND is_new('asset_cursor', a.lastUpdatedTime)
```

- **Always alias** the DM source and use the alias when referring to `space` (e.g., `a.space = ...`). After a JOIN, an unqualified `space = ...` is ambiguous and may bind to the wrong side.
- Prefer Toolkit variables (`'{{instance_space}}'`) over hard-coded space external IDs.

## Edge transformations — `startNode`, `endNode`, `type`

Construct edge endpoints with `node_reference()` and supply the correct space for each side. The `type` of the edge is also a node reference:

```sql
SELECT
    concat(a.externalId, '_to_', b.externalId)                  AS externalId
  , node_reference('{{instance_space}}', a.externalId)          AS startNode
  , node_reference('{{instance_space}}', b.externalId)          AS endNode
FROM ...
```

In the YAML, `edgeType.space` and `edgeType.externalId` must match the `type` constructed in SQL.

---

# Delete Transformation Patterns

CDF deletes via transformations work by writing rows to a destination with `conflictMode: delete`. The query must return the set of `externalId`s (and `space`) to remove. The safest and most idiomatic pattern is **"select what exists, anti-join what should exist"**:

## Node deletes — anti-join pattern

```sql
SELECT
    classic.space         AS space
  , classic.externalId    AS externalId
FROM cdf_nodes('{{schema_space}}', 'Asset', '{{model_version}}') classic
LEFT JOIN (
  -- This subquery mirrors the upsert transformation exactly
  SELECT
      cast(externalId AS STRING) AS externalId
  FROM `{{raw_db}}`.`{{raw_table}}`
) upsert
  ON classic.externalId = upsert.externalId
WHERE classic.space      = '{{instance_space}}'
  AND upsert.externalId IS NULL
```

- The subquery **must** generate `externalId`s the same way the upsert transformation does — any drift causes valid rows to be deleted.
- The outer `classic.space` filter limits the scan and the delete to the correct instance space.

## Edge deletes — scope by `type`

For edges, additionally scope the delete to the specific edge type so deletes do not bleed into unrelated edges in the same view:

```sql
SELECT
    dm.space         AS space
  , dm.externalId    AS externalId
FROM cdf_edges('{{schema_space}}', 'PumpToValve', '{{model_version}}') dm
LEFT JOIN (
  SELECT concat(pump.externalId, '_to_', valve.externalId) AS externalId
  FROM ...
) upsert
  ON dm.externalId = upsert.externalId
WHERE dm.space      = '{{instance_space}}'
  AND dm.type       = node_reference('{{schema_space}}', 'pump_to_valve')
  AND upsert.externalId IS NULL
```

---

# Performance & Memory Optimization

CDF Transformations run on a shared Spark SQL backend. Inefficient queries do not just run slowly — they can be killed by the executor for OOM or shuffle blowups, or starve other transformations sharing the cluster.

## Execution model — what JOINs actually cost

Understanding how the engine fetches data is essential for writing efficient transformations:

- **JOINs are evaluated client-side in Spark.** Both sides of a JOIN are fetched into the executor and then joined. There is no server-side push-down of the join itself into RAW or DM.
- **Only the table guarded by `is_new()` is read incrementally.** Every other source — including the right side of a JOIN — is fetched in full on every run.
- **Implication:** a "simple" two-table JOIN against a 100M-row reference table reads all 100M rows every scheduled run, regardless of how few rows changed. Combine with point 2 below to keep these queries cheap:
  1. Drive the JOIN from the smallest, most-frequently-changing source, with `is_new()` on it.
  2. Replace lookup JOINs with `node_reference()` whenever the JOIN's only purpose is to resolve a direct relation (see Common SQL Patterns).
  3. If neither is possible and the lookup table is large, the transformation may be a poor fit for the service — push the join upstream into the extractor or a Function.

## JOIN optimization and query planning

- **Pick the right JOIN type.** `INNER` only when you require a match on both sides; `LEFT` when the right side is optional. Audit every JOIN for accidental many-to-many fanout that multiplies row counts.
- **Lookup / single-value tables.** When joining against a tiny config table (often one row), prefer a CTE materialized once or a scalar subquery rather than a full JOIN — the planner sometimes generates wasteful plans for these.
- **`UNION ALL` over the same source.** If a query reads the same source table twice and `UNION ALL`s the results, restructure with `CASE WHEN` over a single scan instead.
- **Subquery vs. JOIN.** For very small lookups a correlated subquery may be cheaper; for anything substantial prefer an equi-join.

## Memory-efficient query design

- **Never `SELECT *` in the final destination query.** Project only the columns actually needed, especially from large RAW tables.
- **Push filters early.** Apply WHERE clauses to the source CTE, not after JOINs or aggregations. The planner usually does this, but help it by writing it that way.
- **Expensive functions in JOIN keys.** Repeated `CAST()`, `get_json_object()`, or other functions on a JOIN key are evaluated per row on both sides. Materialize them once in a CTE.
- **Aggregations.** Filter the source down before `GROUP BY`. Avoid stacked intermediate aggregations that recompute the same totals.

## Arrays — flatten over nested `array_union`

`array_union()` only takes two arguments. Chaining it for 3+ inputs is unreadable and error-prone. Use `flatten(ARRAY(...))` and `array_distinct()` instead:

```sql
-- AVOID
array_union(array_union(array_union(a, b), c), d)

-- PREFER
array_distinct(flatten(ARRAY(a, b, c, d)))
```

Apply `FILTER`, `transform`, and `array_distinct` at the right stage to keep intermediate collection sizes small.

## Distributed processing & shuffle optimization

CDF Spark SQL distributes work across nodes. Anything that forces data redistribution (a "shuffle") between nodes is expensive — JOINs, GROUP BY, window functions, and `DISTRIBUTE BY` all trigger shuffles.

- **Non-equi joins are the worst offender.** `JOIN ... ON array_contains(arr, value)` (or any non-equality predicate as the JOIN condition) cannot be hash-partitioned. The engine falls back to a broadcast or nested-loop join and shuffles enormous amounts of data. Always prefer:

  ```sql
  -- AVOID
  JOIN other o ON array_contains(t.tagIds, o.tagId)

  -- PREFER: explode then equi-join
  FROM t
  LATERAL VIEW EXPLODE(t.tagIds) AS tagId
  JOIN other o ON o.tagId = tagId
  ```

- **Co-locate JOIN keys.** Joining on a derived/computed key (e.g., `ON cast(...) = cast(...)` or `ON concat(...) = concat(...)`) prevents the planner from hash-partitioning. Materialize the key in a CTE first when possible.
- **`DISTRIBUTE BY` / `CLUSTER BY`.** Use these intentionally to pre-partition data ahead of a heavy join. `CLUSTER BY = DISTRIBUTE BY + SORT BY` and is the preferred form when the join key is known.
- **Partition pruning.** If the source is partitioned on disk (e.g., by `site_acronym`), include a WHERE filter on the partition column early to skip unnecessary scans.
- **Data skew.** A JOIN or GROUP BY key with one value dominating the distribution (e.g., 90% of rows share a value) bottlenecks one worker. Detect with a `SELECT key, count(*) ... GROUP BY key ORDER BY 2 DESC LIMIT 20` probe. Mitigate by salting the key or filtering out the hot value separately.
- **`array_agg` / `COLLECT_SET` scope.** These build a collection on a single node per group. Filter the source as much as possible before grouping so each group's collection stays small.

---

# Reviewing an Existing Transformation

When the user asks you to **review** a transformation (rather than write one), walk through the checklist below in order. For each issue found, report:

1. **Category** (one of the seven below)
2. **Specific line or pattern**
3. **Risk or problem it causes**
4. **Suggested fix**, and whether it is an **isolated SQL fix** or requires a **source-table change** (schema, indexing, partitioning, etc.)

## 1. JOIN optimization and query planning

- INNER vs LEFT JOIN appropriateness; any risk of Cartesian product or many-to-many fanout
- Single-row / lookup tables handled via CTE or scalar subquery
- `UNION ALL` patterns over the same source restructurable as `CASE WHEN`
- Correlated subquery vs. JOIN trade-off appropriate for expected volume
- **JOINs whose only purpose is to resolve a direct relation** — replace with `node_reference('{{instance_space}}', source_key)` to avoid fetching the lookup table entirely

## 2. Memory-efficient query design

- `SELECT *` anywhere — flag it
- Expensive `CAST()`, `get_json_object()`, or custom functions used in JOIN keys or repeatedly in subqueries
- GROUP BY / window functions minimize in-memory data; no unnecessary intermediate aggregations
- WHERE filters pushed early; nothing pulled into memory only to be discarded
- Array construction uses `flatten(ARRAY(...))` over nested `array_union`; `FILTER` / `transform` / `array_distinct` applied at the right stage

## 3. CDF custom function usage

- **`is_new()` placement**
  - Applied to a single "main" driver source
  - `OR is_new(...)` across multiple joined sources flagged (forces full read of each source before join, and stale data from one side joins to new data on the other)
  - Alias / cursor name is descriptive and unique
  - **Predicate pushdown risk** with non-indexed filters — wrap in `CAST(... AS BOOLEAN)` to prevent pushdown
  - **RAW vs DM signature** — `is_new(name, lastUpdatedTime)` for RAW; `is_new(name)` for `cdf_*` sources (date strings are cursor names, not filters)
  - DM `is_new()` runs at least every 3 days or cursor resets
- **`cdf_data_models()` / `cdf_nodes()` / `cdf_edges()`**
  - Correct argument order: `(space, externalId, version[, viewExternalId])`
  - Filters applied as early as possible to shrink graph scan
  - **Mandatory instance-space predicate** present (`alias.space = 'sp_dat_xxx'` or `alias.space = '{{instance_space}}'`)
  - Space predicate is **alias-qualified** to avoid ambiguity after JOINs
- **`node_reference()`** — startNode, endNode, and edge type built with correct spaces
- **Field names with spaces** — wrapped in backticks (`` `Field Name` ``), never quotes

## 4. Incremental processing health

- `is_new()` is on the right source — the one that changes most frequently and logically drives the transformation
- No anti-patterns that force full reads (`OR is_new`, filtering on derived/computed columns before `is_new`)
- No hard-coded field references / CASTs that would silently fail on a schema change
- An obvious backfill path exists (e.g., bump the cursor name, drop `is_new`, swap in a date filter)

## 5. DM / delete transformation patterns

- Delete uses the `cdf_data_models(...) LEFT JOIN (upsert subquery) ... WHERE classic.externalId IS NULL` pattern
- Edge deletes include `WHERE dm.type = node_reference(...)` to scope by edge type
- `space` is derived consistently and used in both the outer DM reference and the JOIN
- `externalId` construction is deterministic and matches the upsert transformation exactly
- **Container ownership** — every selected column maps to a container this transformation owns; overlay transformations don't re-write properties owned by a base transformation, and don't drag unrelated containers into the `requires` chain

## 6. Distributed processing & shuffle optimization

- JOIN and GROUP BY keys are co-located (not computed on the fly)
- No non-equi joins (`array_contains` or other non-equality predicates as the JOIN condition) — recommend `LATERAL VIEW EXPLODE` + equi-join
- `DISTRIBUTE BY` / `CLUSTER BY` used intentionally if a heavy join follows
- WHERE clauses filter on partition columns early
- Data skew risk identified on JOIN / GROUP BY keys
- `array_agg` / `COLLECT_SET` scoped tightly (filtered before grouping)

## 7. Formatting and readability

- Leading commas at the start of column lines
- CTEs used to name complex intermediate steps
- Consistent indentation; wraps within a typical editor width
- Inline comments on complex JOIN conditions and CASE logic
- All SQL keywords uppercase; all CDF functions lowercase

---

# Best Practices

1. **Default `ignoreNullFields: false`** — this overwrites existing values with null when a column is null, which is the correct behaviour when a single transformation owns all properties of an object. Set to `true` only when multiple transformations write to the same node, so they don't overwrite each other's fields with nulls.
2. **Prefer CDF Workflows for orchestration** — trigger and schedule transformations via a Workflow rather than using the built-in transformation schedule. This gives better control over execution order, dependencies between transformations, and observability.
3. **Use `upsert` conflict mode** by default; use `delete` only for cleanup transformations.
4. **Incremental loading** — use `is_new()` for large single-table datasets. For joins, see rule 6 below.
5. **Use Toolkit variables** (`{{instance_space}}`, `{{schema_space}}`, etc.) to avoid hardcoded values across environments.
6. **`is_new()` tracks one checkpoint — be explicit about which timestamp drives it.** When joining multiple tables, filtering on a single table's `lastUpdatedTime` silently misses updates from other tables. Use `GREATEST(a.lastUpdatedTime, b.lastUpdatedTime)` to cover all sources, or use separate `is_new()` calls with distinct variable names and `OR`. When in doubt, a full load with `conflictMode: upsert` is simpler and safer.
7. **Always use distinct variable names for each `is_new()` call** in the same transformation — sharing a name between two calls risks one overwriting the other's checkpoint before the run completes.
8. **Run at least every 3 days** when using `is_new()` with DMS sources, or the service resets and reprocesses all data.
9. **Avoid `SELECT *` in destination queries** — be explicit about columns to prevent schema drift surprises.
10. **`dataset_id()` over hardcoded numeric IDs** — dataset numeric IDs differ between environments.
11. **Test incrementally** — use the preview feature in the Transformations UI before scheduling.
12. **Always include an instance-space filter on DM reads** (`cdf_data_models`, `cdf_nodes`, `cdf_edges`) and alias-qualify it (`a.space = '{{instance_space}}'`) so it survives JOINs unambiguously.
13. **Avoid non-equi joins** — `JOIN ... ON array_contains(...)` and other non-equality predicates prevent hash partitioning and explode the shuffle. Prefer `LATERAL VIEW EXPLODE` + equi-join.
14. **Use `flatten(ARRAY(...))` instead of chained `array_union(...)`** when combining 3+ arrays. `array_union` only takes two arguments and nested calls are unreadable and bug-prone.
15. **Guard `is_new()` against predicate pushdown** — when combining `is_new()` with a non-indexed filter on an DM source, wrap the other filter in `CAST(... AS BOOLEAN)` so the planner does not push it down into DM and trigger a full scan.
16. **Match delete transformations to their upsert** — `externalId` construction, `space`, and (for edges) edge `type` must be identical in both, or deletes will remove valid rows.
17. **Replace direct-relation JOINs with `node_reference()`** — if the JOIN's only purpose is to resolve a relation target, build the reference directly from the source key column instead of fetching the lookup table.
18. **Stagger schedules** — spread cron start minutes across transformations so the cluster doesn't see a thundering herd on the hour. `is_new()` makes frequent schedules cheap, so prefer staggering over making everything hourly-on-the-hour.
19. **Respect container ownership** — each transformation writes only to containers it owns. Overlay transformations should not re-populate properties owned by a base transformation (especially when those properties live on a container that triggers a `requires` chain).
