# Enterprise vs. Solution Data Models

Guidance for layering CDF data models so that solutions can evolve independently of the enterprise/conceptual model. Read alongside `cdf-data-model-structure.md` (containers, views, CDM rules) and `cdf-schema-versioning.md` (version bumps).

## 1. Layering and ownership

Cognite officially structures CDF data models into three delivery layers with clear ownership. The layer names below match `/cdf/dm/dm_guides/dm_designing_scalable_models`.

| Layer | Owner | What it is | What it owns | Medallion analogy |
|-------|-------|-----------|--------------|-------------------|
| **Source** | Data producers | Raw copy of data from a *single* source system | ELT pipelines into CDF, upstream lineage, source-data monitoring | Silver |
| **Enterprise** | Data modelers | Integrated schema across multiple sources; the validated single source of truth | Canonical view/container definitions, governed property names | Gold |
| **Solution** | Data consumers | App- or use-case-specific surface (search, maintenance, analytics, ...) | Solution-shaped views, denormalized for the use case | Application-specific |

**Additional Cognite-governed base:** the CDM / IDM system spaces (`cdf_cdm`, `cdf_idm`) provide externally stable canonical types (`CogniteAsset`, `CogniteTimeSeries`, `CogniteMaintenanceOrder`, ...). Treat these as immutable — the enterprise layer extends CDM/IDM via `implements:`, and solution views can either implement CDM directly (when they need UI/AI integration) or map to enterprise containers.

The enterprise model is **conceptual**. Solution models should not adopt it as their runtime API; they should reference its **containers** and define their own views.

### Simplified two-layer entry point (small scoped work only)

The docs endorse a **simplified layered architecture** as a starting point — "for example, only two layers" — when the full three-layer split would be over-engineered. Situations where two layers can be enough:

- A short proof of value or scoped experiment with a single source and a single consumer.
- A one-off analytics use case where no second consumer is likely.

The docs don't prescribe *which* two layers; both patterns exist in practice (Source → Solution when the integration is trivial, or Enterprise → Solution when a canonical model already exists). Introduce the third layer as soon as a second source or a second consumer needs to reuse the integrated model — the enterprise layer earns its place when it removes duplication across sources or solutions.

> **This is *not* the same thing as a Cognite QuickStart engagement.** A QuickStart delivery is a **three-layer** setup: it typically ingests from multiple sources into an enterprise model and exposes one or more solution models on top. Follow the full three-layer architecture in QuickStart / Expand engagements. The two-layer starter above is for smaller scoped work where the enterprise middle would add complexity without value — and it is also a different concept from the `dp:quickstart` demo-data deployment pack.

### Write-back principle

For write-back use cases (any workflow that lets a user or app write data *into* CDF from a UI or downstream system), the target **must** be a *separate source data model* owned by a data producer. **Never write back directly into the enterprise layer.** The enterprise layer integrates upstream sources; letting writes land there breaks lineage, contradicts ownership, and makes rollbacks impossible. See sec.5 for the shared-instance-space pattern that supports write-back without polluting enterprise data.

## 2. Containers are the durable contract

Versioning applies to **views and data models**, not containers. Containers evolve additively in place; breaking container changes (property removal, type / `list` / `usedFor` / direct-relation target changes) require export → delete → recreate → re-ingest, not a version bump.

This means:

- **Solution views should map to enterprise containers**, not to enterprise views, via `container:` + `containerPropertyIdentifier:`. Mapping to a view re-couples you to that view's lifecycle; mapping to a container only couples you to the container's (additive) shape.
- The enterprise model's job is to define and govern **containers**. Its views are the canonical reading of those containers; solution views are alternative readings.
- See `cdf-schema-versioning.md` for the additive-change rules and migration paths.

## 3. `implements` vs. mapping — pick the right tool per layer

| Direction | Recommended | Why |
|-----------|-------------|-----|
| Enterprise view → CDM/IDM (`cdf_cdm`, `cdf_idm`) | **`implements:`** | Externally stable; gives you Atlas AI, Industry Canvas, Search, Asset Explorer integration for free |
| Solution view → enterprise container | **Mapping** (`container:` + `containerPropertyIdentifier:`) | Decouples solution lifecycle from enterprise view versions |
| Solution view → another solution view | **Avoid `implements:`** | Recreates the coupling problem at a different level |
| Solution view → CDM/IDM | `implements:` only when the solution genuinely needs CDM UI/AI semantics | E.g. a search solution that exposes assets to Asset Explorer needs `implements: CogniteAsset` |

`implements:` is not the enemy — it is the right tool for `cdf_cdm` / `cdf_idm`. It is the wrong tool between your own layers.

## 4. Reverse relations live with their forward relation

Reverse direct relations (`multi_reverse_direct_relation`, `single_reverse_direct_relation`) belong in the model that owns the **forward** relation:

- Forward relation on an enterprise/CDM container (e.g. `CogniteAsset.parent`) → reverse can stay on the enterprise side (e.g. `Tag.children`).
- Forward relation on a solution-specific view (e.g. `WorkOrder.assets`, `Notification.asset`, `TimeSeriesData.assets`) → reverse belongs in the solution that defines it.

Avoid declaring large numbers of solution-shaped reverses on the enterprise asset/tag view. Each reverse adds a coupling point: the enterprise model now depends on the solution's forward property name and target view version.

Pair this with the rules in `cdf-direct-relations.md` (REVERSE-009): reverse on view **A** through **B`.p`** still requires **`p.source` → A** when stored references resolve as **A**.

## 5. Shared instance space, multiple data models

If multiple solution models reference the same logical entity (e.g. a tag), they should share the **same node external ID** in a shared **instance space**, mirroring how functional locations are typically handled:

- One instance space (e.g. `sp_ops_domain_model.Instance`) holds the canonical nodes.
- Each solution's data model lives in its own model space and reads/writes those nodes through its own views.
- Solutions agree on external ID conventions so that joins across solutions stay coherent.

### Source vs. enterprise instance strategy

Cognite's recommendation is to use **separate instance spaces per layer**, where each domain owns the instance space it writes to:

- A PI extractor writes to a source data model backed by an instance space owned by the source domain (e.g. `sp_pi_source.Instance`).
- The enterprise data model reads/derives from source data and writes into a **separate** enterprise-owned instance space (e.g. `sp_enterprise.Instance`).
- Solution models typically map data *within* the enterprise layer without writing back to it (see the write-back principle in sec.1).

**Why separate:** if source and enterprise share an instance space, deleting a source instance also deletes the enterprise instance derived from it, which is almost never what you want. Separation costs more instances but eliminates that data-loss risk and simplifies access control.

**When to combine:** only when you have deliberately traded instance-count savings for the deletion coupling — and the source domain understands that source-side deletes propagate into the enterprise.

### Access control implications

- **Graph access is scoped by *space*, not by data set.** Data sets remain the governance construct for other CDF resource types, but they do **not** control who can read or write instances in DMS. Do not design DMS access control around data sets.
- Two ACLs are relevant to DMS: **`dataModels`** (schemas — spaces, containers, views, data models) and **`dataModelInstances`** (data — nodes and edges). Each has `READ` and `WRITE`.
- **`dataModelInstances:WRITE_PROPERTIES`** lets a principal write property values without allowing them to create or delete instances. Useful for automation that enriches existing nodes without being able to introduce or remove them.
- Autocreate rules for direct-relation targets require **write** access to the target space; nodes in read-only target spaces must exist beforehand.
- Time series and files are dual-permission: `datamodels:read` on `cdf_cdm` for the schema, plus `datamodelinstances:read` on the specific instance space for the data. **Separate instance spaces per site/plant** (e.g. `sp_location_a_instances`) are the primary tool for granular access control.

Together, these justify **schema spaces and instance spaces as separate spaces from the start** — access control is impossible to retrofit later.

## 6. One `CogniteAsset` implementer per data model

CDF UI navigation (Asset Explorer, Industry Canvas) breaks when a data model contains multiple views that `implements: CogniteAsset`. This rule applies **per data model**, not per space:

- Each data model that needs asset semantics defines its **own single** `CogniteAsset` implementer. A search solution model cannot reuse the enterprise `Tag` view if it wants to be self-contained — it should have its own `SearchTag` (or similar).
- Solution models that don't need asset hierarchy semantics should avoid `CogniteAsset` entirely — use `CogniteDescribable` plus direct relations.

See also `cdf-data-model-structure.md` → *CogniteAsset — Single Implementation Only*.

## 7. Search solution is not a hub

A search-oriented data model is a **solution**, not a shared canonical model:

- It contains the data its use cases need, denormalized for query speed.
- It does not own enterprise semantics or canonical relations.
- Other solutions should not depend on the search model's views.
- Treat search-specific views as disposable: they can be re-shaped or re-versioned without coordinating across the org.

## 8. Templates and blueprints

Reference solution models (e.g. a tag-centric template, a maintenance template) are useful **starting points** — not mandatory implementations. To make a template safe to fork:

- Expose a **configuration surface** (Toolkit variables, generator config, `default.config.yaml`) so adopters customize without editing view YAML directly.
- Mark which parts are **stable contract** (do not edit) vs. **scaffolding** (free to delete or extend) in the README.
- Ship a **NEAT validation notebook** or equivalent so adopters can verify their fork still passes the same checks.
- Keep the template up to date with CDM/IDM evolution; tag releases against CDM/IDM versions.

## 9. Versioning strategy

With the layering above, versioning becomes mostly local:

- **Containers:** unversioned; additive only. Breaking changes require migration.
- **Enterprise views:** bump version on breaking change; coordinate updates across all consumers in the same PR (see `cdf-schema-versioning.md`).
- **Solution views:** bump independently; consumers are typically known apps within one team.
- **Data model `version`s:** keep separate variables in `default.config.yaml` per model (e.g. `dm_enterprise_version`, `dm_search_version`) so they can move independently.

## 10. Consumer tracking (interim, until observability exists)

CDF does not yet expose per-consumer version usage. Until it does, treat the following as policy, not best practice:

- Every data model README has a **`consumers:` section** listing known apps and their pinned versions. Update it in the same PR that bumps a version.
- Never delete a deployed view version without a written **deprecation window** (90 days minimum is a reasonable default). Mark deprecated views in their `description`.
- Treat `cdf deploy --dry-run` diff output as the **consumer-impact preview**. Any property removal in a view is a breaking change for unknown consumers — assume someone uses it.
- Property removal in a deployed container is destructive; coordinate any such migration explicitly with all known consumers and the data owner.

## 11. Denormalize at solution boundaries, normalize in the enterprise

Shape and ownership are different decisions. The enterprise model stays normalized to preserve canonical semantics; solution models — especially search — denormalize aggressively for query speed and AI/UI consumption.

Examples:

- A `TimeSeriesData` solution view can merge PI and OPC UA properties (e.g. `pi_*` / `opcua_*` prefixes) into one wide view, even if the enterprise model keeps PI and OPC UA properties separate.
- A `Files` solution view can flatten document → revision → file into one view, even if the enterprise model keeps them as three nodes.
- A search solution view can copy parent/area/system context onto each tag for one-shot retrieval.

See `cdf-data-model-structure.md` → *Normalization vs Denormalized Access Views* for the trade-offs.

## 12. Quick decision checklist

Before adding a view, ask:

- [ ] Is this view **enterprise** (canonical, governed) or **solution** (use-case-specific)?
- [ ] If solution: am I mapping to enterprise **containers**, not implementing enterprise **views**?
- [ ] If I'm using `implements:`, is the target in `cdf_cdm` / `cdf_idm`, or is this introducing cross-layer coupling I'll regret?
- [ ] If I'm adding a reverse relation, is the **forward** relation defined in the same model? If not, should the reverse move to the model that owns the forward?
- [ ] Does this data model already have a `CogniteAsset` implementer? If so, am I about to create a second one?
- [ ] If this is a solution view that needs to share entities with another solution, am I using a shared instance space and a stable external ID convention?
- [ ] Is there a `consumers:` entry that needs updating because of this change?
- [ ] Are the source and enterprise layers using **separate** instance spaces owned by their respective domains, or is the coupling risk documented and accepted?
- [ ] For write-back workflows, does the write target a *source* data model (not the enterprise layer)?
- [ ] Are schema spaces and instance spaces separate, with distinct groups holding `dataModels` vs `dataModelInstances` capabilities?