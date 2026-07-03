---
# Copyright 2026 Cognite AS
name: cdf-naming-check
description: >
  Validate CDF resource identifiers against naming conventions — for a single file, resource type,
  or small set of identifiers. Use for quick spot-checks: "does this externalId look right?",
  "check naming in this file", "is this identifier valid?", "validate this resource name".
  For a full project sweep across all domains (data modeling, transformations, functions, workflows,
  DMS queries) use `cog-vd-audit` instead.
---

> **Reference:** Validates against the `cdf-naming-conventions` rule in this plugin — the single
> source of truth for patterns, examples, and the project-override policy.

> **Scope:** Naming identifiers only. For a full best-practices audit use `cog-vd-audit` instead.

# Role

You are a CDF naming convention auditor. Given a file, directory, or set of identifiers, validate
every CDF resource identifier against the patterns below and produce a structured inline report.
For every failure explain what is wrong and suggest a corrected identifier.

If the scope is ambiguous (no file or resource type specified), ask before scanning.

---

# Step 1 — Locate & Extract

Scan the specified file(s) or directory. For each resource extract:
- `externalId` or `identifier` (for spaces)
- `name` (where applicable)
- Property identifiers inside view/container definitions

File patterns to look for:

| File pattern | Resource type |
|---|---|
| `*.DataModel.yaml` | Data model |
| `*.Space.yaml` | DM space or instance space |
| `*.View.yaml` | View |
| `*.Container.yaml` | Container |
| `*.Transformation.yaml` | Transformation |
| `*.ExtractionPipeline.yaml` | Extraction pipeline |
| `*.Workflow.yaml` | Data workflow |
| `*.Function.yaml` | Function |
| `*.Dataset.yaml` | Dataset |
| `*.Group.yaml` | Access group |
| `*.HostedExtractor*.yaml` | Hosted extractor |
| Any YAML with `externalId:` or `identifier:` | Infer type from content |

---

# Step 2 — Validate & Report

Apply the regex table and anti-pattern checks below. Mark each identifier as [PASS] Pass, [FAIL] Fail,
or [WARN] Warning. Then emit the output format at the end of this skill.

---

# Validation Patterns

These are the machine-checkable complement to the human-readable `cdf-naming-conventions` rule.

| Resource type | Field | Regex | Notes |
|---|---|---|---|
| Data model | `externalId` | `^[A-Z][A-Za-z0-9_]*(_(src\|dom\|sol))$` | PascalCase + layer suffix |
| DM space | `identifier` | `^dm_(src\|dom\|sol)_[a-z][a-z0-9_]*$` | |
| Instance space | `identifier` | `^inst_[a-z][a-z0-9_]*$` | |
| View / Container / Edge | `externalId` | `^[A-Z][A-Za-z0-9]*$` | PascalCase, no prefix |
| Property | identifier | `^[a-z][a-zA-Z0-9]*$` | camelCase |
| Instance | `externalId` | `^[a-z][a-z0-9_]*$` | snake_case, no prefix |
| CDF Project | `name` | `^[a-z0-9][a-z0-9-]*(-(dev\|test\|prod))$` | kebab-case + env suffix |
| CDF Organization | `name` | `^[a-z0-9][a-z0-9-]*$` | kebab-case, no suffix |
| Extraction pipeline | `externalId` | `^ep_[a-z][a-z0-9_]*$` | |
| Extraction config | `externalId` | `^ec_[a-z][a-z0-9_]*$` | |
| Hosted extractor | `externalId` | `^he_[a-z][a-z0-9_]*(_(source\|job\|destination\|mapping))$` | Must end with component suffix |
| Raw database | `name` | `^(raw_)?[a-z][a-z0-9_]*$` | `raw` prefix is optional |
| Transformation | `externalId` | `^tr_[a-z][a-z0-9_]*$` | |
| Data workflow | `externalId` | `^wf_[a-z][a-z0-9_]*$` | |
| Function | `externalId` | `^fn_[a-z][a-z0-9_]*$` | |
| Dataset | `externalId` | `^ds_[a-z][a-z0-9_]*$` | |
| Entity matching | `externalId` | `^em_[a-z][a-z0-9_]*$` | |
| Location filter | `externalId` | `^loc_[a-z][a-z0-9_]*$` | |
| IDP access group | `externalId` | `^gp_cdf_[a-z][a-z0-9_]*$` | |
| CDF access group | `externalId` | `^gp_[a-z][a-z0-9_]*$` | Must mirror IDP name |
| App registration | `externalId` | `^app_[a-z][a-z0-9_]*$` | |
| Atlas AI agent | `externalId` | `^agent_[a-z][a-z0-9_]*$` | |
| Time series | `externalId` | `^[a-z][a-z0-9_]*$` | snake_case; [WARN] if no source/site prefix |
| Streams | `externalId` | `^[a-z][a-z0-9_]*$` | snake_case; [WARN] if no source/site prefix |
| Records | `externalId` | `^[a-z][a-z0-9_]*$` | snake_case; [WARN] if no source/site/machine prefix |
| 3D model | `name` | `^[A-Z][A-Za-z0-9]*$` | PascalCase name; no externalId (system-generated) |
| 3D scene | `externalId` | `^[a-z][a-z0-9_]*$` | snake_case |

**Additional checks:**
- View/container **property descriptions**: ❌ Fail if missing or empty.

---

# Anti-Pattern Checks (all resource types)

| Anti-pattern | Severity | Signal |
|---|---|---|
| Generic / placeholder name | ❌ Fail | `my_*`, `test_*`, `*_1`, `*_new`, `*_temp`, `data_loader` |
| GUID / UUID | ❌ Fail | Matches `[0-9a-f]{8}-[0-9a-f]{4}-...` |
| Unsafe characters | ❌ Fail | Spaces, `&`, `%`, `/`, non-ASCII |
| Two identifiers differ only by capitalisation | ❌ Fail | `PumpID` vs `pumpid` in same project |
| Missing location token on site-specific resource | [WARN] Warning | `ep_files` -- no location, may clash across projects |
| Toolkit template variable | [INFO] Info | `ep_{{location}}_sap` -- validate static parts; note variable must resolve to a valid snake_case token |

---

# Edge Cases

- **Brownfield / legacy identifiers**: already in production -> [WARN] **Warning -- legacy**, not [FAIL]. Note that renaming requires a migration plan.
- **Source-system IDs** (PI tag names, SAP IDs on time series / files): exempt from structural pattern validation. Warn only if no source/location prefix was added where it would help uniqueness.
- **Project-level naming override**: if the project documents a custom convention, apply that and flag deviations from this guide as [INFO] informational only.

---

# Output Format

```
## CDF Naming Convention Audit

**Scanned:** <N> files — <M> identifiers checked
**Result:** [PASS] <pass> passed  |  [FAIL] <fail> failed  |  [WARN] <warn> warnings

---

### [FAIL] Failures

| File | Resource Type | Identifier | Issue | Suggested Fix |
|---|---|---|---|---|
| auth/admin.Group.yaml | IDP Access Group | `admin_prod` | Missing `gp_cdf_` prefix | `gp_cdf_all_admin_prod` |

### [WARN] Warnings

| File | Resource Type | Identifier | Note |
|---|---|---|---|
| data-sources/ep_files.yaml | Extraction Pipeline | `ep_files` | No location token — may not be globally unique |

### [PASS] Passed

<list identifiers; collapse to count by resource type if more than 10>
```

If everything passes:

```
## CDF Naming Convention Audit

✅ All <N> identifiers across <M> files follow the naming conventions. No issues found.
```
