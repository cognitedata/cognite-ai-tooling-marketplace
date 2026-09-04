# CDF Data Model Structure

## Property Limits

`usedFor: node` and `usedFor: record` containers are **different resource shapes**, not the same container with a relaxed cap. `node` containers back the graph / data-model layer (they have views, constraints, indexes, and are queried through the DM API). `record` containers are a separate high-volume event/log primitive queried directly, with **no views, constraints, or indexes**. The property caps below follow from that difference ΓÇö see also the *`usedFor` ΓÇö pick the right shape* section below.

- **100 properties per `usedFor: node` container** ΓÇö this is the **default** for a CDF project and can be raised by Cognite Support if there's a strong reason (avoid making it a habit).
- **1,000 properties per `usedFor: record` container** ΓÇö the higher cap reflects the record shape's purpose (wide event/log rows), not looser governance on the same container type.
- Properties cannot be removed from deployed containers. Plan ahead.
- Use `additionalProperties` (type: json) as a catch-all for overflow or rarely-queried fields. Storing "not first-class" data in a `json` property is officially supported: if a field is never used for filter/sort/index, there is no reason to promote it to a scalar property.
- Count properties before adding new ones. If near the limit, evaluate which properties are truly needed.
- See `cdf-data-model-limits.md` for the full defaults table (spaces, containers, views, instances, list sizes, etc.).

## Normalization vs Denormalized Access Views
Use CDM/IDM as the normalized semantic base, but design consumer-facing access to be denormalized by default.

### Default Strategy (Strong Recommendation)
- Keep canonical semantics in CDM/IDM (`implements` + `requires`) so model meaning stays standard and interoperable.
- Prefer **one extended, denormalized access view per entity** that includes commonly needed properties for read/use cases.
- Add properties to that extended view instead of creating multiple sibling views for the same entity unless there is a hard requirement.
- Optimize for "easy to read in one query": reduce multi-hop joins for common analytics, search, and application reads.

### When to Split into Separate Views
Only create separate views when one of these is true:
1. Access control requires strict property isolation between audiences
2. Property count/shape limits make a single view impractical
3. Lifecycle/versioning differs significantly between property sets
4. Performance tests show a measurable benefit from separation that indexing/denormalization cannot solve

### Anti-Pattern
- Do not create fragmented "technical" views that force consumers to join many views just to read core business context.

### Where to denormalize
Denormalize at **solution boundaries**, not in the enterprise model. The enterprise layer stays normalized to preserve canonical semantics (one container per concept, clean relations). Solution views ΓÇö especially search and AI-facing views ΓÇö flatten aggressively (e.g. merging PI + OPC UA properties with `pi_*` / `opcua_*` prefixes, or flattening document ΓåÆ revision ΓåÆ file into one view) for one-shot retrieval. See `cdf-enterprise-vs-solution.md` sec.11.


## Extending CDM Types
When extending a Cognite Data Model type (CogniteAsset, CogniteTimeSeries, etc.):

**View**: use `implements` to inherit from the CDM view:
```yaml
implements:
- space: cdf_cdm
  externalId: CogniteAsset
  version: v1
  type: view
```

**Container**: use `requires` constraints to declare the CDM dependency:
```yaml
constraints:
  assetPresent:
    constraintType: requires
    require:
      space: cdf_cdm
      externalId: CogniteAsset
      type: container
```

### Query Optimization via `requires`

For any view that spans multiple containers, `requires` container constraints are **mandatory** ΓÇö the Cognite docs state this outright: without them, the query engine cannot prune the joins triggered by a `hasData` filter across the view's mapped containers, and **query performance is typically unacceptable**. If a view exposes properties from or has relationships to a CDM/IDM type, the underlying **`usedFor: node`** container **must** declare a `requires` constraint on that type's container. This does **not** apply to **`usedFor: record`** containers (they have no `constraints` block).

The query planner uses `requires` to short-circuit the `significantHasDataFiltering` debug notice ΓÇö when it knows that data in container B implies data in container A, it only needs to check A. See `cdf-data-model-indexes.md` and `cdf-dms-queries` for the debug-notice details.

**Important:** Do **not** add unrelated `requires` constraints just to silence optimization warnings. A `requires` constraint is an ingest-time dependency and will make writes fail unless the required container is also present on the same node.

For example, a file can validly exist without any time series relation. Therefore, do **not** require `cdf_cdm:CogniteTimeSeries` on a file extension container unless file nodes are truly guaranteed to also be time series nodes in your domain (which is usually not the case).

Common required constraints by pattern:

| View implements | Container must require (in addition to direct parent) |
|----------------|------------------------------------------------------|
| `CogniteMaintenanceOrder` | `cdf_cdm:CogniteActivity` (activity is the CDM base) |
| `CogniteNotification` | `cdf_cdm:CogniteActivity` |
| `CogniteOperation` | `cdf_cdm:CogniteActivity` |
| `CogniteDescribable` (view-level) | `cdf_cdm:CogniteDescribable` (container-level) |

Always run `cdf build` or deploy to check for "not optimized for querying" warnings, then add only semantically valid `requires` constraints.

## `usedFor` ΓÇö pick the right shape

The `usedFor` field controls which instance types a container can populate:

| `usedFor` | Meaning | Constraints & indexes | View required |
|---|---|---|---|
| `node` | Standard entity data (assets, equipment, work orders) | Yes ΓÇö use `constraints` (including `requires`) and `indexes` | Yes |
| `edge` | Properties on edges (the relationship carries data) | Yes | Optional |
| `all` | Container usable on both nodes and edges | Yes | Depends on use |
| `record` | High-volume event/time-stamped rows (alarms, logs) | **No** ΓÇö omit `constraints` and `indexes` | No (queried directly) |

- Prefer dedicated `node` or `edge` ΓÇö **`all` is more expensive to ingest** than a container that commits to one instance type.
- `record` containers are queried directly against the container, not through the data model API. Record containers also support **1,000 properties** (versus 100 for node); see `cdf-data-model-limits.md`.
- Optional pointer-style properties on a record container (e.g. a `tag` reference on an alarm row) stay as plain properties ΓÇö no `requires`, no btree tuning.

## CogniteAsset ΓÇö Single Implementation Per Data Model
Only **one view per data model** should `implements: CogniteAsset`. Multiple views implementing CogniteAsset causes UI navigation problems in CDF applications (e.g., IndustryCanvas, Asset Explorer). If you need multiple asset-like entities, have one primary asset view implement CogniteAsset and use direct relations from the others.

This rule applies **per data model**, not per space. A solution data model that needs asset semantics must define its own single `CogniteAsset` implementer; it should not reuse the enterprise model's implementer if the solution is meant to be self-contained. Solution models that don't need asset hierarchy semantics should avoid `CogniteAsset` entirely ΓÇö use `CogniteDescribable` plus direct relations. See `cdf-enterprise-vs-solution.md` sec.6.

## IDM Types for Work Management
Prefer implementing the **IDM types** from `cdf_idm` for work order, notification, and operation entities. This gives standardized semantics (`mainAsset`, `type`, `status`, `priority`, etc.) that CDF applications expect.

| Entity | View implements | Container requires | Key IDM properties |
|--------|----------------|-------------------|-------------------|
| WorkOrder | `cdf_idm:CogniteMaintenanceOrder` | `cdf_idm:CogniteMaintenanceOrder` | `mainAsset`, `type`, `status`, `priority`, `priorityDescription`, `operations` (reverse) |
| Notification | `cdf_idm:CogniteNotification` | `cdf_idm:CogniteNotification` | `asset`, `type`, `status`, `priority`, `priorityDescription`, `maintenanceOrder` |
| WorkOrderOperation | `cdf_idm:CogniteOperation` | `cdf_idm:CogniteOperation` | `maintenanceOrder`, `mainAsset`, `phase`, `status`, `sequence`, `mainDiscipline`, `numberOfMainDiscipline`, `personHours` |

Source these properties from the IDM container (`space: cdf_idm`), not from custom containers.

If `implements` is not used, you must satisfy the minimum mapping requirements in **Minimum CDM/IDM Mapping Without `implements`** so the view is still recognized correctly by CDF clients.

## CDM Properties ΓÇö Never Duplicate
When a view `implements` a CDM type, properties like `name`, `description`, `tags`, `aliases` (CogniteDescribable), `sourceId`, `source`, `sourceCreatedTime` (CogniteSourceable), and `startTime`, `endTime`, `scheduledStartTime` (CogniteSchedulable) must always reference the CDM container (`space: cdf_cdm`). Never create custom container properties that shadow or duplicate these ΓÇö always source them from the CDM container.
- Verify canonical CogniteDescribable mappings to avoid copy-paste mistakes: `name -> name`, `description -> description`, `labels/tags -> tags`, `aliases -> aliases`.

## Minimum CDM/IDM Mapping Without `implements`
If you do not use `implements` for CDM/IDM types, map the minimum required CDM/IDM container properties so CDF clients can classify the view correctly.

### Asset Classification (Search/Canvas)
- To be treated as an Asset, map at least one CogniteAsset container property in the view (for example `type`).
- Search detects mapped containers in the view; if the view maps to CogniteAsset container properties, it is treated as an asset and gets the asset icon.

### Time Series Classification
- To be treated as a Time Series, map `isStep` and `type` from CogniteTimeSeries container properties.
- When these mappings are present, the time series icon appears automatically.

### Event Classification (Charts)
- To be treated as an Event and show in Charts, map both `startTime` and `endTime` from CogniteSchedulable container properties and ensure both are populated with data.
- Also map `assets` from CogniteActivity container properties for activity-to-asset context.

### Property ExternalId Stability
- Do not change view property externalIds when mapping container properties.
- You may change the display `name`, but the view property externalId and `containerPropertyIdentifier` should normally match the mapped container property externalId.
- Explicit aliases are allowed when they improve domain clarity or avoid naming collisions (for example `labels` mapped to `CogniteDescribable.tags`). When aliasing, keep `containerPropertyIdentifier` mapped correctly and document the alias intent in the property description.

## Container-View Alignment
- Every `usedFor: node` container should have a corresponding `*.View.yaml` exposing its properties.
- `usedFor: record` containers do **not** need a view ΓÇö they are not part of the data model's view layer.
- Every container property exposed in a view must use `containerPropertyIdentifier` matching the container's property key.
- Don't expose container properties in a view without a clear use case ΓÇö views are the query API.
- Every direct relation in a view **must** have a `source` block specifying the target view with `space`, `externalId`, `version`, and `type: view`. This tells the system which view to resolve the referenced node through, enabling typed navigation in the UI.

```yaml
# Required for all direct relation properties in views
source:
  space: "{{space}}"
  externalId: Tag
  version: "{{dm_version}}"
  type: view
```

Only omit `source` when the target is intentionally polymorphic (e.g., `classSpecific` pointing to multiple equipment class views) or when no view exists for the target node type.

For reverse traversals, align forward `source`, reverse host view, and actual stored edge targets (parent vs satellite); details in **`cdf-direct-relations.md`** (*ForwardΓÇôreverse pairing and anchor view*, NEAT-DMS-CONNECTIONS-REVERSE-009).

## Descriptions
- Every container and view must have a top-level `description`.
- **Every view property must have a `description` field.** Properties without descriptions make the model opaque to AI tools, search engines, and developers. A property named `pi_compDev` is meaningless without "PI compression deviation threshold".
- Keep descriptions concise (one sentence) and domain-specific ΓÇö not dictionary definitions.
- Container and view descriptions for the same entity should be consistent.
- Top-level description must match the entity represented by `externalId` (avoid cross-entity copy-paste, e.g., a Well container described as a Shortfall event).
- No stray quotes or YAML escaping artifacts in description values.
- Never copy-paste descriptions between views without adapting context (e.g., "tags the time series is related to" is wrong in a Files view).
- Explain abbreviations on first use (e.g., "SPN priority" ΓåÆ "SPN (Safety Priority Number) priority score").
- Reverse relations should describe the relationship direction (e.g., "Notifications that reference this failure mode").
- Top-level view descriptions should be specific about what the view exposes, not generic ("A view that represents information about tags" is bad; "Core tag properties including area, facility, system, and relationships to equipment classes" is good).

## Property Aliasing ΓÇö `labels` vs `tags`
When the CogniteAsset view is named `Tag`, the inherited `tags` property from CogniteDescribable (text-based labels) creates a naming conflict. Expose it as `labels` in views and always use `labels` in transformations and queries ΓÇö never `tags`, which will be confused with the Tag view or its direct relations.

## Naming and Semantic Integrity
- Treat property identifiers as long-lived API contracts: avoid typos and accidental renames (for example `isoloations` instead of `isolations`).
- Ensure `sourceCreated*` and `sourceUpdated*` properties have matching names and descriptions (no semantic swaps).
- For any alias mapping, ensure the `description` clearly states the underlying container property to reduce ambiguity for users and AI tools.

## Container-View Property Completeness
- Every property in a `usedFor: node` container **should** be exposed in the corresponding view. Unmapped container properties are invisible to the query API and AI tools.
- When adding properties to a container, always add corresponding view properties in the same change.
- Audit regularly: compare container `properties:` keys against view `containerPropertyIdentifier` values to find gaps.

## CDM Container References ΓÇö Match the Implemented Type
When a view `implements` a CDM type (e.g., `CogniteFile`) and exposes CDM-defined properties like `assets`, the `container` reference in the view **must** point to the correct CDM container for that type. Common mistake: copy-pasting `CogniteTimeSeries` as the container when the view actually implements `CogniteFile`.

```yaml
# WRONG ΓÇö Files view using CogniteTimeSeries container
assets:
  container:
    space: cdf_cdm
    externalId: CogniteTimeSeries  # Bug: should be CogniteFile
    type: container
  containerPropertyIdentifier: assets

# CORRECT ΓÇö container matches the implemented type
assets:
  container:
    space: cdf_cdm
    externalId: CogniteFile
    type: container
  containerPropertyIdentifier: assets
```

Always verify that CDM container references match the `implements` declaration at the top of the view.

### Explicit Verification Rule: Container vs Source
- For CDM-defined properties exposed in a view, the `container` must reference the matching CDM container for the implemented type (e.g., `implements: CogniteAsset` -> `container.externalId: CogniteAsset`; `implements: CogniteFile` -> `container.externalId: CogniteFile`).
- For direct relation properties, the `source` should reference the most specific view in this data model when available (your extended/implemented view), not a generic CDM ancestor view.
- In short: **CDM container for property storage semantics; specific model view for relation navigation semantics.**

## Structural Audit Checklist
When reviewing or modifying a data model, verify:

1. **Property mapping**: Every container property has a corresponding view property (for `usedFor: node` containers)
2. **Descriptions**: Every view property has a `description` field; no copy-paste errors
3. **Direct relation sources**: Every `type: direct` container property exposed in a view has a `source` block (unless polymorphic)
4. **Index coverage**: Every `type: direct` container property has a btree index
5. **Reverse relation validity**: Every `through.identifier` matches an actual property name in the referenced view
6. **No `source` on non-direct**: `source` blocks only appear on properties backed by `type: direct` container properties
7. **Property count**: No container exceeds 100 properties
8. **CDM sourcing**: Properties from CogniteDescribable/CogniteSourceable/CogniteSchedulable reference CDM containers, not custom ones
9. **Top-level fields**: Node containers include `constraints` and `indexes`; record containers omit them; views have `space`, `externalId`, `description`, `version`, `properties`
10. **CDM container match**: Every CDM property's `container.externalId` matches the type declared in `implements` (not copy-pasted from another view)
11. **View-to-data-model coverage**: Every `*.View.yaml` file in the module is referenced by at least one `*.DataModel.yaml` file (no orphaned views)
12. **No unrelated requires**: Every `constraintType: requires` reflects a true entity dependency, not a workaround (e.g., no `Files -> CogniteTimeSeries` unless files are guaranteed to be time series nodes)
13. **Alias clarity**: Any property alias (for example `labels <- tags`) is intentional, documented, and keeps correct `containerPropertyIdentifier` mapping
14. **Naming integrity**: No typo-like property identifiers and no semantic swap between `sourceCreated*` and `sourceUpdated*` fields
15. **Canonical CDM mapping**: CogniteDescribable mappings are semantically correct (`aliases` is not mapped to `description`, etc.)
16. **Description-entity consistency**: Top-level container/view description semantically matches its own `externalId` entity
17. **No reserved identifiers**: External IDs, property names, and space names avoid the reserved lists (see [`cdf-data-model-limits.md` ΓåÆ *Reserved values*](./cdf-data-model-limits.md#reserved-values))
18. **External ID length**: Every instance external ID is <= 255 characters and contains no null bytes
19. **No EAV pattern**: Data is modeled with first-class properties (or `json` overflow), not as attribute-value node explosions
20. **Frequently-updated fields isolated**: `lastViewed`/`lastUpdated`/counter-style fields do not live on wide, heavily-indexed containers
21. **`usedFor` fits the shape**: dedicated `node` or `edge` rather than `all` unless there's a real reason
22. **Polymorphism style is intentional**: structural (`implements` + `hasData`) or nominal (`filter` on `node.type`); connection-only views set an explicit filter

## Container sizing ΓÇö wide, narrow, and the EAV anti-pattern

Container width has real query-performance consequences. Two failure modes:

- **Containers too wide + heavily indexed + frequently updated.** Every write to any indexed property rewrites the whole row in the underlying store, so a "one big container" design combined with frequently-updated fields like `lastViewed`, `viewCount`, or `lastInspection` causes container/index bloat, ingestion latency, and eventually `HTTP 408` timeouts on queries. **Split frequently-updated properties into their own container** so their writes don't touch the heavy indexes.
- **Containers too narrow ("one property per container").** Every consumer query needs to JOIN across many containers; the query planner's statistics degrade, and composite indexes (which must live in a single container) become impossible.

**HTTP 408 signals a sizing/index/statistics problem** ΓÇö treat 408s as evidence that a container layout should be reviewed.

**Do not use the Entity-Attribute-Value (EAV) pattern in DMS.** The Cognite docs call this out explicitly as an anti-pattern:

- Each new property multiplies instance counts, since every property becomes its own instance node.
- Adding a couple of properties can increase stored data by 600 %+ due to per-instance system metadata.
- Every query becomes a many-to-many-to-many JOIN, defeating the planner's statistics.
- Indexes cannot be granular ΓÇö they end up covering everything, so every property becomes slow to query.

Model your data with **first-class properties**. If a value is never used for filter/sort/index, put it in a `json`-typed property rather than promoting it to its own container. Follow the Zipf-distribution reality: the top few properties carry most of the query load, and everything else can live in JSON without penalty.

## Polymorphism ΓÇö two approaches

CDF supports two polymorphism styles; pick the one that fits your view's shape.

### Structural (`implements` + implicit `hasData` filter)

The default. When view `Pump` implements `Equipment`, `Pump`'s implicit filter is `hasData` across both the `Pump` and `Equipment` containers. Filtering the parent `Equipment` view returns anything with data in `Equipment`; filtering `Pump` returns anything with data in both. "If it looks like a duckΓÇª"

**Breaks down** when a view has only connection properties (reverse relations, edge connections) and no backing containers ΓÇö there's nothing to `hasData`-filter on. Use nominal filtering instead.

### Nominal (`filter` on `node.type`)

Explicitly assign a type node to each instance and filter the view on that type:

```yaml
filter:
  equals:
    property: ['node', 'type']
    value: { space: 'types', externalId: 'pump' }
```

To list a parent view that has multiple subtypes, use `in`:

```yaml
filter:
  in:
    property: ['node', 'type']
    values:
      - { space: 'types', externalId: 'pump' }
      - { space: 'types', externalId: 'valve' }
```

Nominal filtering is required for connection-only views and useful when the same node needs to be reached through multiple views that share containers.

### Type nodes and the `types` space

Every instance has a `type` property (a direct relation to a **type node**). Best practice is to keep type nodes in a dedicated `types` space so they're easy to govern and access-control separately. A type node cannot be deleted while any instance still points to it ΓÇö plan the lifecycle before deploying.

## View filters ΓÇö default is `hasData`

If a view does not declare a `filter`, DMS applies an implicit `hasData` filter across the containers the view maps. This means:

- A view that only maps `ContainerA` returns any node with data in `ContainerA`.
- A view that maps `ContainerA` and `ContainerB` returns nodes with data in **both**.
- A connection-only view (no mapped containers) needs an explicit `filter` ΓÇö usually `equals: [node, type]` ΓÇö or it returns nothing meaningful.

Set an explicit filter when the default `hasData` semantics don't match the entity you want to expose.

## Edges span spaces

An edge lives in one space but can link nodes in other spaces. When designing multi-solution models with a shared instance space:

- Store the edge in the space that owns the relationship type (often a solution model), even if `startNode` and `endNode` live in a shared canonical instance space.
- Deleting a node cascades edge deletion. When a node has many incoming/outgoing edges, **delete the edges first**, then the node ΓÇö otherwise the cascade can time out.

## Native reference property types

In addition to node `direct` relations, container properties can point at legacy Cognite API resources using native reference types:

| Native reference type | Points to |
|---|---|
| `TimeSeries` | A single time series in the Time Series API |
| `File` | A file stored via the Files API |
| `Sequences` | A sequence in the Sequences API |

Use these when the referenced object is not itself a graph node and belongs in a non-DMS API. Prefer `direct` relations to `CogniteTimeSeries` / `CogniteFile` **nodes** when the reference should participate in the graph (search, canvas, atlas, etc.).

## `implements` ΓÇö property precedence with conflicts

When a view implements multiple other views that define the same property identifier, CDF resolves the conflict by topologically sorting the `implements` graph:

- The **direct view** wins over any parent.
- Beneath that, the order in the `implements: [...]` array matters: **later entries win over earlier ones** at the same level.

If view `B implements [C, D]` and both `C` and `D` define property `x`, then `B` sees `D.x` (later entry wins). Reordering to `implements: [D, C]` makes `C.x` win. Avoid conflicts where possible ΓÇö they're a maintenance trap for anyone extending your view later.

## AI-facing modeling and property naming

Views and properties are consumed by Atlas AI agents, natural-language search, and document parsing. Following these rules materially improves accuracy:

- **Prefer human-readable names.** Avoid abbreviations and source-system codes as property identifiers (`pi_compDev` bad, `piCompressionDeviation` better). If you must keep an abbreviated identifier, spell it out on first use in the description ("SPN (Safety Priority Number) priority score").
- **Type-level docstrings should include usage context.** Don't just say what the type is ΓÇö say what it's used for and what abstractions it covers. A single `Event` type used for both work orders and alerts should say so, so the LLM knows the same type answers both kinds of questions.
- **Every property description should include an example value** where the value shape isn't obvious. "ID from the source system. Example: 21003104" is much more useful to an LLM than "ID from the source system".
- **Constrained values ΓåÆ enum types.** If a property has a fixed set of legal values (`Open` / `Closed` / `Released`, priority codes, status codes), model it as a container `enum` property (up to 32 values per enum). This makes the value space discoverable to AI without prompting.
- **Document parsing depends on property naming.** The confidence score returned by `document_parser` is based on how closely view property names match the field names in source documents (datasheets, PDFs). Design a view against a representative document *before* batch-parsing ΓÇö property names that don't resemble the document's field names produce low confidence.

## Reserved identifiers

CDF reserves specific values for container/view `externalId`, container property names (plus enum value names), and space external IDs. Avoid them when naming any of these resources ΓÇö the API will reject reserved names outright.

For the canonical lists, see [`cdf-data-model-limits.md` ΓåÆ *Reserved values*](./cdf-data-model-limits.md#reserved-values). Kept in a single location to prevent drift.

## External ID length

- **Max 255 characters** for any instance (node/edge) external ID.
- **No null bytes.**
- Applies inside the space ΓÇö uniqueness is `(space, externalId)`.

## Required Top-Level Fields
- **`usedFor: node` containers:** `space`, `externalId`, `description`, `properties`, `constraints`, `indexes`, `usedFor`
- **`usedFor: record` containers:** `space`, `externalId`, `description`, `properties`, `usedFor` ΓÇö do **not** include `constraints` or `indexes`
- **Views:** `space`, `externalId`, `description`, `version`, `properties` (plus `implements` when extending CDM)

## Data Models
- Data model files list all views that form the model's public API.
- Each view entry under `views:` may include **`name`** (human-readable display label). Some validators (e.g. NEAT) flag missing names on these references even when the `*.View.yaml` already defines `name` ΓÇö keep the data model entry in sync with the view file.
- Every view in a data model file must exist as a `*.View.yaml` file.
- **Every `*.View.yaml` file in the module must be referenced by at least one `*.DataModel.yaml` file.** Orphaned view files are never deployed and silently drift from the live model. When adding a new view, always add a corresponding entry in the data model file. When auditing, compare the set of `*.View.yaml` filenames (minus extension) against all `externalId` values with `space: "{{space}}"` across the module's `*.DataModel.yaml` files ΓÇö any view file not matched is orphaned.
- Use `{{dm_version}}` for version and `{{space}}` for space to support multi-environment deployment.
- For **view version bumps**, **container migration**, and **uniqueness** constraints, see `cdf-schema-versioning.md`.

## View-Only UX Modules
- In UX/exploration modules that primarily expose views (and rely on containers in other spaces), keep property mappings semantically aligned with the source-domain views.
- Propagate mapping bug fixes across mirrored views (for example `aliases -> aliases`, corrected source metadata labels, and corrected `containerPropertyIdentifier` values) to avoid drift between domain and UX layers.
