# Schema changes, view versions, and data model references

Distilled for Toolkit YAML workflows. For **index and btree rules**, use `cdf-data-model-indexes.md` (max **10** btree indexes per `usedFor: node` container).

## View `version` bumps

When you change a view in a **breaking** way (delete/remap properties, change types, change `implements`, change relation shape), bump **`version`** in the view YAML and update **every** reference to that view:

1. The **`*.View.yaml`** file's top-level `version`.
2. Any other view that references this view in **`source`**, **`through`**, or **`implements`** (match `version` to the deployed view).
3. Every **`*.DataModel.yaml`** entry under `views:` for this `externalId` / `space`.

Missing one of these produces stale GraphQL / deployment errors.

## Data model `views:` entries

Each list item is a **view reference** only: `space`, `externalId`, `version`, `type: view`. Cognite Toolkit treats any other field (e.g. `name`) as invalid on those entries. Put the human-readable **`name`** on the **`*.View.yaml`** resource itself, not on the data model's `views:` list.

## Containers (long-lived schema)

Containers are **not** versioned like views ΓÇö they are the **durable contract** between enterprise and solution layers. Treat deployed container schema as **additive** where possible:

- **Avoid** deleting properties or changing property **type**, **`list`**, **`usedFor`**, or direct-relation target ΓÇö these are destructive or disallowed paths that require migration (export, delete container, recreate, re-ingest).
- **Safer:** add new properties, adjust **name** / **description**, add nullable fields, add indexes/constraints per current CDF rules (see indexes reference).

Because containers are unversioned, mapping a solution view to an enterprise **container** (`container:` + `containerPropertyIdentifier:`) decouples the solution from enterprise view version churn. Mapping (or `implements:`) to an enterprise **view** re-couples you to that view's lifecycle. See `cdf-enterprise-vs-solution.md` sec.2ΓÇôsec.3.

Confirm current CDF rules with **`SearchCogniteDocs`** or `cdf build` before advising a specific migration.

## `requires` / index lifecycle

Adding or changing **`requires`** and **indexes** is processed **asynchronously** in CDF. The create endpoint returns immediately; the actual validation or index build happens in the background against existing data. If ingest or queries behave oddly right after a deploy, check instance/constraint state in the API or UI before assuming misconfiguration.

### Background validation states

Constraints, indexes, and property-level rules (nullability, `maxListSize`, `maxTextSize`) each expose a read-only **`state`** field on the returned container. Three possible values:

- **`current`** ΓÇö validated / built successfully. Ready for use.
- **`pending`** ΓÇö background work is still running. Check back later.
- **`failed`** ΓÇö pre-existing data violates the constraint, or blocks the index from building. **Constraints are still enforced on new data even when the state is `failed`** ΓÇö old rows just aren't validated retroactively, and failed indexes cannot speed up queries.

For property-level constraints (nullability, size bounds) the state fields live under `constraintState` on the property object (`nullability`, `maxListSize`, `maxTextSize`).

### Retrying a failed constraint or index

CDF does not automatically re-validate after `failed`. Once you've cleaned up the offending data, trigger a re-scan explicitly:

- **`indexes/retry`** ΓÇö retry a failed btree/inverted index build.
- **`constraints/retry`** ΓÇö re-validate `requires` or `uniqueness` constraints that had invalid pre-existing data.
- **`properties/retry`** ΓÇö re-validate nullability / size-bound constraints on properties that had invalid pre-existing data.

Each of these triggers a full scan against the container's data ΓÇö **use them sparingly**, especially on large containers.

## Uniqueness

Use **`constraintType: uniqueness`** on business keys when one container must enforce a unique combination of scalar properties ΓÇö in addition to btree indexes used for lookup and filters.

## Breaking change patterns

### Containers

Containers are not versioned. Each operation falls into one of three categories: non-breaking and allowed, breaking but allowed, or breaking and disallowed.

To perform a **disallowed** operation, use the expand-and-contract pattern:
1. Create a new container with the desired spec.
2. Re-ingest the property data into the new container for the relevant instances.
3. Create new view versions that map to the new container, and optionally new data model versions.
4. Once old views are no longer in use, delete the old containers and views.

After deleting, recreating, re-ingesting, or creating a parallel container, recreate any existing view-to-container mappings.

| Operation | Breaking | Allowed | Notes |
|---|---|---|---|
| Change name | No | Yes | Metadata only |
| Change description | No | Yes | Metadata only |
| Change `usedFor` | N/A | **No** | Not allowed |
| Add property | No | Yes | Identifier must not already be in use |
| Delete property | N/A | **No** | Not allowed |
| Add `requires` or check constraint | No | Yes | Applies to new values only, not pre-existing |
| Add unique constraint | N/A | Yes | Only during initial container creation |
| Change constraint | N/A | **No** | Not allowed |
| Delete constraint | No | Yes | Allowed |
| Add new index | No | Yes | Allowed |
| Delete index | No | Yes | Allowed |
| Change index | N/A | **No** | Not allowed |
| Change property: nullable ΓåÆ non-nullable | Yes | Yes | May break ingestion clients |
| Change property: non-nullable ΓåÆ nullable | N/A | **No** | Not allowed |
| Change property: `autoIncrement` | N/A | **No** | Not allowed |
| Change property: `defaultValue` | No | Yes | Applies to new values only |
| Change property: `description` | No | Yes | Metadata only |
| Change property: `name` | No | Yes | Metadata only |
| Change property: `type` | N/A | **No** | Not allowed |
| Change (text) property: `list` state | N/A | **No** | Not allowed |
| Change (text) property: collation | N/A | **No** | Not allowed |
| Change (primitive) property: `list` state | N/A | **No** | Not allowed |
| Change (direct relation) property: target container | N/A | **No** | Not allowed |

### Views

Views are versioned. Bump `version` and cascade all references for any breaking change (see sec.View `version` bumps above).

| Operation | Breaking | Notes |
|---|---|---|
| Change name | No | Metadata only |
| Change description | No | Metadata only |
| Change filter | No | Changes results, not the form |
| Change `implements` | Yes | May break clients |
| Change version | Yes | Even a version bump with no other changes is breaking |
| Add nullable property | No | Safe if no collision with inherited properties and no new mapped container introduced |
| Add non-nullable property | Yes | Clients may depend on its existence |
| Delete property | Yes | |
| Change property type | Yes | Requires version bump |
| Change container reference for base property | No | Allows remapping to new container without forcing a version bump on consumers |
| Change source hint | Yes | Equivalent to changing property type |
| Change type of relation | Yes | |
| Change direction of relation | Yes | |
| Change source of relation | Yes | |

### Data Models

| Operation | Breaking | Allowed | Notes |
|---|---|---|---|
| Change name | No | Yes | Metadata only |
| Change description | No | Yes | Metadata only |
| Add a view | No | Yes | Non-breaking if the new view's `externalId` doesn't conflict with existing views |
| Remove a view | Yes | Yes | Clients may depend on the view's existence |
| Replace a view | Yes | Yes | Typically done to update the version of a view used by the data model |
| Change version | Yes | Yes | A version bump with no other changes is still breaking |
| Change space | N/A | **No** | Not supported |
