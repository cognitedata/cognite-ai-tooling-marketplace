# CDF Data Modeling — Limits and Reserved Values

Consolidated defaults from the Cognite docs (`/cdf/dm/dm_reference/dm_limits_and_restrictions`). These are **defaults** for a CDF project — some can be raised on request via Cognite Support. Use this file as a quick lookup during design and audits; consult the docs if a specific number matters for a decision.

## Schema and instance limits

| Resource | Default limit |
|---|---:|
| Spaces per project | 100 |
| Containers per project | 1,000 |
| Container properties (total across project) | 25,000 |
| Container properties per `usedFor: node` container | 100 |
| Container properties per `usedFor: record` container[^usedfor] | 1,000 |
| Enum values per container property | 32 |
| View versions per project (total) | 2,000 |
| View versions per view | 100 |
| View properties per view (`node`) | 300 |
| View properties per view (`record`) | 1,000 |
| Implemented views per view | 10 |
| Data model versions per project (total) | 500 |
| Data model versions per data model | 100 |
| Views per data model | 100 |
| Live instances per project | 5,000,000 |
| Soft-deleted instances per project | 10,000,000 |

[^usedfor]: `node` and `record` are different resource shapes, not the same container with a different cap — `record` containers have no views, constraints, or indexes and are queried directly rather than through the DM API. See `cdf-data-model-structure.md` → *`usedFor` — pick the right shape*.

## Property value size limits

| Property type | Max size |
|---|---|
| `text` | 128,000 bytes (UTF-8) |
| `json` | 40,960 bytes (UTF-8) |

## List property size caps

- **Direct relation lists:** default cap 100 items.
- **Other list types:** default cap 1,000 items.
- **Customizable via `maxListSize`** — you can only *raise* the cap after container creation, never lower it.
- **Absolute maxima for `maxListSize`:**
  - `int32` (with btree index): 600
  - `int64` (with btree index): 300
  - Other types (any list including `text[]`, `direct[]` without btree): 2,000
- For **btree eligibility** on list properties, see `cdf-data-model-indexes.md` (list-type btree restrictions).

## Instance identifiers

- **External ID:** max 255 characters, no null bytes. Uniqueness is `(space, externalId)`.
- **Soft-delete grace period:** 3 days. Soft-deleted instances count toward the live-instance quota; they can be returned by `/sync` with a non-null `deletedTime`. Cannot be restored.

## API concurrency

Concurrency is applied project-wide, not per client. Excess requests get `429 Too Many Requests` (official SDKs retry with exponential backoff).

| Operation type | Default concurrent operations |
|---|---:|
| Instance `apply` | 4 |
| Instance `delete` | 2 |
| Instance `query` | 8 |

**Transformations may use up to 75 % of the concurrency budget**; the remaining 25 % is reserved for other clients.

## Reserved values

Never use these as identifiers — CDF rejects them.

### Reserved `externalId` for containers and views
`Boolean`, `Date`, `File`, `Float`, `Float32`, `Float64`, `Int`, `Int32`, `Int64`, `JSONObject`, `Mutation`, `Numeric`, `PageInfo`, `Query`, `Sequence`, `String`, `Subscription`, `TimeSeries`, `Timestamp`.

### Reserved container property names
`createdTime`, `deletedTime`, `edge_id`, `endNode`, `extensions`, `externalId`, `lastUpdatedTime`, `node_id`, `project_id`, `property_group`, `seq`, `space`, `startNode`, `tg_table_name`, `version`.

### Reserved enum value names
`"true"`, `"false"`, `"null"`.

### Reserved space external IDs
`cdf`, `constraint`, `dms`, `edge`, `edge_source`, `history`, `identifier_by_space`, `index`, `mapping`, `node`, `node_source`, `pg3`, `project`, `property`, `property_group`, `property_group_constraint`, `property_group_index`, `shared`, `space`, `system`, `version_info`.

## Monitoring current usage

- **CDF UI:** Data models → *Storage → See all* to view project-specific limits and current usage.
- **API:** `datamodels/statistics` (project-wide) and space-scoped `datamodels/statistics` (per space).

## Raising limits

- **Data modeling schema and instance limits:** contact [Cognite Support](https://cognite.zendesk.com/hc/en-us/requests/new) with a justification.
- **Records/streams capacity beyond defaults:** describe what you need in record count, storage volume (GB), and retention per stream. Route the request via one of:
  1. Your Cognite Customer Success or Solution Architect contact, if you have one.
  2. Otherwise, a [Cognite Support](https://cognite.zendesk.com/hc/en-us/requests/new) ticket mentioning **"Records / Streams custom capacity"**.

  See the docs' *On-request capacity* section for the upper range Cognite can provision. Request capacity **before** creating a stream — per-stream limits are fixed at creation and cannot be raised later.

## Records and streams (summary)

For projects using records/streams, additional per-stream limits apply. Consult the docs (`/cdf/dm/dm_reference/dm_limits_and_restrictions` → *Records and streams*) for template-specific caps (records per stream, throughput, `lastUpdatedTime` filter interval, retention). This skill does not cover records/stream design in depth.
