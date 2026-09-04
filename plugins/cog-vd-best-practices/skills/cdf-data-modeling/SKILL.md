---
name: cognite-data-modeling
description: CDF data model design patterns for containers, views, and CDM/IDM extensions in YAML. Use when creating or modifying data model YAML files, extending CogniteCore or IDM types, designing containers and views, adding indexes or direct relations, auditing data models, or deploying via Cognite Toolkit.
---

> **Naming conventions:** Apply the `cdf-naming-conventions` rule from this plugin for all resource identifiers (spaces, containers, views, properties, building blocks, etc.).

# CDF Data Modeling

Read this skill first for orientation, then read the relevant reference file:
- `references/cdf-data-model-structure.md` -- containers, views, CDM/IDM extensions, `usedFor`, polymorphism, view filters, native reference types, AI-facing modeling, container sizing + EAV anti-pattern, reserved identifiers, audit checklist
- `references/cdf-data-model-indexes.md` -- btree vs inverted indexes, default-indexed base properties, btree size bounds, composite index ordering, requires-is-mandatory, cross-links to debug notices
- `references/cdf-direct-relations.md` -- direct relation patterns, edge connection properties (`single_edge_connection` / `multi_edge_connection`), reverse relations, source specificity, forwardΓÇôreverse pairing (REVERSE-009)
- `references/cdf-schema-versioning.md` -- view version bumps, data model `name` on view refs, container change safety, background validation states + retry endpoints
- `references/cdf-enterprise-vs-solution.md` -- Source / Enterprise / Solution layering with ownership + medallion mapping, two-layer starter, write-back principle, mapping vs. `implements:`, source/enterprise instance-space strategy, access-control (spaces, not data sets), reverse-relation ownership, consumer tracking
- `references/cdf-data-model-limits.md` -- full defaults table (spaces, containers, properties, views, versions, instances, list/text/json sizes, API concurrency, reserved names)

**Related skills in this plugin:**
- `cognite-dms-queries` ΓÇö writing efficient DMS `/list`, `/query`, `/search`, and `/sync` queries; debug notices; `/list` sort pitfalls; `hasData` mechanics; latency variability.
- `cognite-transformation` ΓÇö SQL transformations that write into data models (uses `is_new(lastUpdatedTime)` as its cursor pattern ΓÇö see the cross-link note in that skill).

For querying and fetching data from a model, use the `cognite-dms-queries` skill.

## 1. The Three-Layer Model

| Layer | YAML file type | Purpose |
|-------|---------------|---------|
| **Container** | `*.Container.yaml` | Physical property storage; defines types; `usedFor: node` adds constraints and indexes for graph/query semantics |
| **View** | `*.View.yaml` | Logical query API; maps container properties, declares relations |
| **Data Model** | `*.DataModel.yaml` | Public API surface; lists which views are exposed |

Every `usedFor: node` container needs a matching view. Every view needs an entry in a data model file. `usedFor: record` containers are queried directly ΓÇö no view needed. **Do not** add `constraints` or `indexes` on `usedFor: record` containers (no graph-node ingest rules or btree tuning for that shape).

## 2. Enterprise vs. Solution Layering

CDF data models stack in three layers. Pick the right tool per layer:

| Layer | Recommended pattern |
|-------|---------------------|
| Enterprise view ΓåÆ CDM/IDM (`cdf_cdm`, `cdf_idm`) | **`implements:`** ΓÇö gives Atlas AI, Industry Canvas, Search, Asset Explorer for free |
| Solution view ΓåÆ enterprise **container** | **Mapping** (`container:` + `containerPropertyIdentifier:`) ΓÇö decouples solution lifecycle from enterprise view versions |
| Solution view ΓåÆ another solution view | Avoid `implements:` ΓÇö recreates coupling at a different level |

**Containers are the durable contract.** Views are versioned; containers are not. Map solution views to enterprise *containers*, not enterprise *views*. Reverse direct relations belong in the model that owns the **forward** relation. See `cdf-enterprise-vs-solution.md` for the full layering, shared-instance-space pattern, template guidance, and consumer-tracking policy.

## 3. Extending CDM/IDM Types

For **`usedFor: node`** containers: extend via `implements` in the view and `requires` in the container. **`usedFor: record`** containers skip `constraints` / `requires` (see sec.1).

```yaml
# View ΓÇö inherit semantics
implements:
  - space: cdf_cdm
    externalId: CogniteAsset
    version: v1
    type: view

# Container ΓÇö declare ingest-time dependency
constraints:
  assetPresent:
    constraintType: requires
    require:
      space: cdf_cdm
      externalId: CogniteAsset
      type: container
```

**CDM properties are never duplicated.** `name`, `description`, `tags`, `aliases`, `sourceId`, `startTime`, `endTime` etc. always map to CDM containers (`cdf_cdm`), never to custom containers.

## 4. Key Design Rules

- **One CogniteAsset implementer per data model** ΓÇö multiple views implementing CogniteAsset breaks CDF UI navigation. This applies *per data model*, not per space: a solution model that needs asset semantics must define **its own** single `CogniteAsset` implementer, not reuse the enterprise one.
- **Default 100 properties per `usedFor: node` container** (1,000 for `usedFor: record`, which is a *different* resource shape ΓÇö no views/constraints/indexes, queried directly; see `cdf-data-model-structure.md` ΓåÆ *`usedFor` ΓÇö pick the right shape*). Plan ahead; properties cannot be removed after deployment. The default can be raised via Cognite Support, but treat it as a design constraint, not a target. Plan overflow strategy (`additionalProperties` JSON, secondary container, separate view) **before** modeling, not after.
- **Max 10 indexes per container** (`usedFor: node` only) ΓÇö index strategically (see `cdf-data-model-indexes.md`).
- **Containers are unversioned and additive** ΓÇö breaking container changes (property removal, type / `list` / `usedFor` changes) require migration, not a version bump. Views and data models carry the version semantics.
- **Every direct relation in a view must have a `source` block** ΓÇö omit only for intentionally polymorphic relations.
- **Every view property must have a `description`** ΓÇö unnamed properties are opaque to AI tools and search. Prefer human-readable identifiers over abbreviations; include an example value when the shape isn't obvious (see AI-facing modeling in `cdf-data-model-structure.md`).
- **`requires` constraints are mandatory for multi-container views** ΓÇö without them, the query planner cannot short-circuit the joins triggered by `hasData` filters, and query performance is typically unacceptable at scale.
- **Never model with EAV (entity-attribute-value).** First-class properties for the fields you filter/sort/index; `json` overflow for everything else. See `cdf-data-model-structure.md` ΓåÆ *Container sizing ΓÇö wide, narrow, and the EAV anti-pattern*.
- **Use `{{space}}` and `{{dm_version}}` template variables** ΓÇö never hardcode space or version values. Use separate version variables per data model (e.g. `dm_enterprise_version`, `dm_search_version`) so they can move independently.

## 5. IDM Work Management Types

Prefer `cdf_idm` types for work orders, notifications, and operations:

| Entity | View implements | Key properties |
|--------|----------------|----------------|
| WorkOrder | `cdf_idm:CogniteMaintenanceOrder` | `mainAsset`, `type`, `status`, `priority` |
| Notification | `cdf_idm:CogniteNotification` | `asset`, `type`, `status`, `priority` |
| WorkOrderOperation | `cdf_idm:CogniteOperation` | `maintenanceOrder`, `mainAsset`, `phase`, `status` |

## 6. Quick Audit Checklist

Before deploying or reviewing a data model:
- [ ] Every `usedFor: node` container property is exposed in the view
- [ ] Every view property has a `description`
- [ ] Every `type: direct` property has a btree index (unless `maxListSize > 300`)
- [ ] Every `type: direct` in a view has a `source` block (unless polymorphic)
- [ ] Every `through.identifier` in a reverse relation matches an actual indexed property
- [ ] Forward `source` on that property matches the view type instances actually store (see `cdf-direct-relations.md` ΓÇö reverse pairing and ingest alignment)
- [ ] Reverse relations live in the model that owns the **forward** relation (not piled onto the enterprise asset/tag view)
- [ ] Solution views map to enterprise **containers**, not `implements:` enterprise views
- [ ] Only one view in this data model `implements: CogniteAsset`
- [ ] No container exceeds 100 properties or 10 indexes; overflow strategy (`additionalProperties` etc.) is in place
- [ ] Every `*.View.yaml` appears in a `*.DataModel.yaml` (no orphaned views)
- [ ] CDM property containers match the `implements` declaration (no copy-paste mismatch)
- [ ] Every multi-container view has the `requires` chain it needs so the query planner can short-circuit `hasData` ΓÇö no missing `requires`
- [ ] No `requires` constraints added for unrelated containers
- [ ] `usedFor: record` containers have no `constraints` or `indexes` sections
- [ ] Frequently-updated fields (counters, `lastViewed`, `lastInspection`) are isolated from heavily-indexed containers
- [ ] No EAV pattern ΓÇö data is first-class properties or `json` overflow, not attribute-value node explosions
- [ ] External IDs, property names, and space names avoid the reserved lists (see [`cdf-data-model-limits.md` ΓåÆ *Reserved values*](references/cdf-data-model-limits.md#reserved-values))
- [ ] Source and enterprise layers use **separate** instance spaces (or the coupling risk is explicitly documented)
- [ ] README has a `consumers:` section listing known apps + pinned versions (updated in this PR if a version bumped)

## 7. Schema, versioning, and validators

- **View `version`:** For breaking view changes, bump `version` in the view YAML **and** update every other view / data model that references that view's `version` (see `cdf-schema-versioning.md`).
- **Data model `views:`:** Only `space`, `externalId`, `version`, `type` ΓÇö no extra fields (Toolkit warns on e.g. `name`). Display names belong on `*.View.yaml`.
- **Containers:** Unversioned. Prefer additive schema changes; property removal and type / `usedFor` changes on deployed containers need a documented migration path ΓÇö do not treat containers as freely mutable.
- **Per-model versions:** Keep separate `dm_*_version` variables per data model so enterprise and solution models can bump independently.
- **Consumer tracking:** Until CDF exposes per-consumer version usage, treat the README `consumers:` section + a written deprecation window (90 days minimum) as policy. Never delete a deployed view version without it. See `cdf-enterprise-vs-solution.md` sec.10.
- **Broader architecture:** For the full Source ΓåÆ Enterprise ΓåÆ Solution layering, mapping vs. `implements:` decisions, shared instance spaces, and template/blueprint patterns, see `cdf-enterprise-vs-solution.md`.

## 8. When to Check Docs

Use the `SearchCogniteDocs` MCP tool for:
- Specific CDM/IDM property lists and container schemas
- Cognite Toolkit YAML syntax and deployment commands
- New CogniteCore versions or IDM type additions
- `cdf build` warning messages and how to resolve them

## 9. When to Stop and Ask the Developer

Do not autonomously make these decisions ΓÇö pause and present the options with a recommendation instead:

| Situation | Why it needs a human |
|---|---|
| Choosing between Source / Enterprise / Solution layer for a new container | Determines ownership, write-back rules, and long-term maintenance responsibility |
| Deciding the `space` for a new schema or instance space | Space names are permanent and affect access control across all environments |
| Bumping a view or data model version on a deployed model | Downstream consumers may break; the developer must confirm the deprecation window |
| Picking which CDM/IDM type to `implements` | Wrong base type loses CDF UI integration (Asset Explorer, Atlas AI, etc.) and is hard to undo |
| Removing or migrating a container property | Requires an expand-and-contract migration; data loss risk if done wrong |
| Choosing between direct relations and edges when the relationship may need properties later | Edges add complexity and cost and cannot be changed back without migration |
| Setting the `maxListSize` on a `direct[]` relation | Affects btree indexing eligibility and query performance trade-offs |
| Defining a new instance space or merging source/enterprise instances | Irreversible data-organisation decision with access control consequences |
| Adding a reverse relation to an enterprise or CDM view | Couples the enterprise model to a solution's forward property name ΓÇö developer must accept that dependency |
| Choosing between a single wide container and multiple contextual containers | Affects the 100-property limit, query scope, and AI tool accuracy |
| Any schema change on a container in production with known consumers | Breaking or not ΓÇö the developer must validate impact on live data pipelines |
