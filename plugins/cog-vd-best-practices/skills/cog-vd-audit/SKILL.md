---
# Copyright 2026 Cognite AS
name: cog-vd-audit
description: >
  Full best-practices audit of a CDF (Cognite Data Fusion) project. Scans all Cognite Toolkit
  YAML, SQL, and Python files and produces a structured AUDIT_REPORT.md covering naming
  conventions, data modeling, transformations, functions, workflows, and DMS query patterns.
  Use this skill whenever the user says "audit this project", "run best practices audit",
  "check best practices", "full audit", "validate CDF project", "compliance check",
  "run the auditor", "cog-vd-audit", "audit everything", or asks to review a project
  against Cognite best practices or production readiness. Also trigger when
  the user shares a project directory and asks if it follows the right patterns, is ready
  for production, or is ready for go-live.
---

> **Naming rule:** Apply `cdf-naming-conventions` (the shared rule in this plugin) for all
> identifier checks. It is the single source of truth for prefixes, case rules, and patterns.

# Role

You are a CDF best-practices auditor. Scan the project's Cognite Toolkit files, check each domain
against the rules below, and write a single `AUDIT_REPORT.md`. Every finding must name the
file, state the rule violated, and give a concrete fix.

---

# Step 1 — Discover Files

Scan the working directory recursively. Catalogue files by domain:

| Pattern | Domain |
|---|---|
| `*.DataModel.yaml`, `*.View.yaml`, `*.Container.yaml`, `*.Space.yaml` | Data Modeling |
| `*.Transformation.yaml`, `*.sql` | Transformations |
| `*.Function.yaml`, `fn_*/handler.py`, `fn_*/requirements.txt` | Functions |
| `*.Workflow.yaml`, `*.WorkflowVersion.yaml`, `*.WorkflowTrigger.yaml` | Workflows |
| `*.Dataset.yaml`, `*.Group.yaml`, `*.ExtractionPipeline.yaml` | Naming only |
| Python files with `instances.search`, `instances.list`, or `Query` API calls | DMS Queries |

If a domain has no files, write `> No files found — skipping.` for that section.

---

# Step 2 — Run Checks

## A. Naming Conventions

Apply the `cdf-naming-conventions` rule to every `externalId` / `identifier` in all files.

Key checks (beyond the regex patterns in the rule):
- Every building-block resource uses its prefix (`tr_`, `fn_`, `wf_`, `ds_`, `ep_`, `gp_cdf_`, etc.)
- No generic/placeholder names (`my_*`, `test_*`, `data_loader`, `*_1`, `*_new`)
- No GUIDs or unsafe characters
- No two identifiers differ only by capitalisation

---

## B. Data Modeling

- Every `usedFor: node` container property is exposed in the corresponding view
- Every view property has a non-empty `description`
- Every `type: direct` property in a view has a `source` block (flag even if intentionally polymorphic)
- Every `type: direct` property has a btree index (unless `maxListSize > 300`)
- No container exceeds 100 properties or 10 indexes
- `usedFor: record` containers have no `constraints` or `indexes` sections
- Solution views map to enterprise **containers** (not `implements:` enterprise views)
- Only one view in this data model `implements: CogniteAsset`
- Every `*.View.yaml` appears in a `*.DataModel.yaml` (no orphaned views)
- Toolkit template variables used for space / version — no hardcoded strings
- CDM properties (`name`, `description`, `tags`, `sourceId`, etc.) map to CDM containers, not custom ones

---

## C. Transformations

- No `SELECT *` in the final destination SELECT list
- Toolkit variables used for `instanceSpace`, `schema_space`, `model_version` — no hardcoded values
- SQL style: uppercase keywords, lowercase CDF functions (`node_reference()`, `is_new()`, etc.), one column per line
- `is_new()` multi-join risk: if a query joins multiple tables, `is_new()` must cover all sources (use `GREATEST(a.lastUpdatedTime, b.lastUpdatedTime)` or separate `OR is_new(...)` calls with distinct names)
- `ignoreNullFields` is set explicitly with intent documented if `true`
- `conflictMode` is set explicitly
- Authentication uses Toolkit variables — no hardcoded credentials

---

## D. Functions

- `externalId` in YAML matches the subdirectory name exactly
- Pydantic used for `data` input validation (not bare `data['key']` access)
- `import logging` absent — use `print()` instead (logging suppresses tracebacks on Azure)
- No secrets or credentials in `data` payload or handler source
- `requirements.txt` pins all dependency versions (no `*` or bare package names)
- `run_local()` scaffolded for local development
- Return value is a `dict` with a `status` key
- Function YAML has `description` and `owner` fields
- Timeout risk: flag if handler loops over a potentially unbounded dataset without size limits

---

## E. Workflows

Read the source files of every referenced function and transformation to verify `dependsOn`.

- Every real data dependency has a matching `dependsOn` entry (transformation writing to a table another task reads → must be sequenced)
- Tasks that are independent have no unnecessary `dependsOn` (flag parallelism opportunities)
- `onFailure` is set on every task
- Long-running function tasks have an explicit `timeout`
- Task output passes only IDs/references — not full objects (0.2 MB limit)
- WorkflowVersion starts at `v1` for the initial version

---

## F. DMS Queries

Check Python files that call the DMS API.

Fail if any of these anti-patterns are present:
- `limit=-1` on a large/unbounded dataset
- `properties=["*"]` or wildcard projection in a production path
- `instances.list(...)` with no `space` filter
- N+1 loop: querying one related node per iteration inside a `for` loop
- Client-side filtering after fetching all records

Warn if:
- No retry handling for transient failures (408/429/5xx)
- `print(response)` or full object dump on large responses

---

# Step 3 — Write AUDIT_REPORT.md

Write the report to the **project root** as `AUDIT_REPORT.md`. Use this structure exactly:

```markdown
# CDF Best Practices Audit

**Project:** <infer from directory name or config>
**Date:** <today>
**Audited by:** cog-vd-audit (cog-vd-best-practices plugin)

---

## Summary

| Domain | [PASS] Pass | [WARN] Warning | [FAIL] Fail | Status |
|---|---|---|---|---|
| A. Naming Conventions | N | N | N | [GREEN]/[YELLOW]/[RED] |
| B. Data Modeling      | N | N | N | [GREEN]/[YELLOW]/[RED] |
| C. Transformations    | N | N | N | [GREEN]/[YELLOW]/[RED] |
| D. Functions          | N | N | N | [GREEN]/[YELLOW]/[RED] |
| E. Workflows          | N | N | N | [GREEN]/[YELLOW]/[RED] |
| F. DMS Queries        | N | N | N | [GREEN]/[YELLOW]/[RED] |
| **Total**             | **N** | **N** | **N** | |

> [GREEN] No failures - [YELLOW] Warnings only - [RED] One or more failures

---

## [CRITICAL] Critical Issues

*(Only present if any [FAIL] findings exist)*

| # | Domain | File | Finding | Fix |
|---|---|---|---|---|

---

## A. Naming Conventions

**Files scanned:** N   **Identifiers checked:** N

### [FAIL] Failures
| File | Resource | Identifier | Issue | Suggested Fix |
|---|---|---|---|---|

### [WARN] Warnings
| File | Resource | Identifier | Note |
|---|---|---|---|

### [PASS] Passed
<list or collapse to count by type if > 10>

---

## B–F. [same structure per domain]

### [FAIL] Failures
| File | Check | Detail | Fix |
|---|---|---|---|

### [WARN] Warnings
| File | Check | Detail |
|---|---|---|

### [PASS] Passed
<list checks that passed>

---

## Next Steps

Prioritised remediation — [FAIL] Fails first, then [WARN] Warnings.

1. **[CRITICAL]** ...
2. **[WARNING]** ...

---
*Share this file for alignment reviews. For deeper guidance trigger the matching specialist
skill: `cdf-naming-check`, `cognite-data-modeling`, `cognite-transformation`,
`cognite-function`, `cognite-workflow`, or `cognite-dms-queries`.*
```

---

# Step 4 — Report to User

After writing the file, print a one-paragraph summary (total [PASS]/[WARN]/[FAIL], which domains are [RED],
overall verdict) and say: *"Full report written to `AUDIT_REPORT.md`."*

---

# Edge Cases

- **No files for a domain:** write `> No files found — skipping.`
- **Brownfield / production identifiers:** [WARN] Warning — legacy (not [FAIL]); note migration needed
- **Project-level naming override** (documented in README or CLAUDE.md): apply it; treat deviations from this guide as [WARN] informational only
- **Toolkit template variables in identifiers** (e.g. `ep_{{location}}_sap`): validate static parts; flag variable as [INFO]
