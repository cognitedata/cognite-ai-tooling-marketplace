# cog-vd-best-practices

Expert guidance and best-practice enforcement for CDF projects, maintained by Cognite. Covers Data Modeling, Transformations, Functions, Workflows, and Data Modeling Service queries.

## What's included

| Skill | Description |
|---|---|
| `cdf-data-modeling` | Data model design patterns for containers, views, and CDM/IDM extensions in YAML |
| `cdf-dms-queries` | Search-first, production-safe query patterns for Cognite DMS, including pagination, traversal, and retries |
| `cdf-function` | Guidance on CDF Functions — handler signatures, secrets, schedules, local development, and deployment |
| `cdf-transformation` | CDF Transformation SQL on the Spark backend — destinations, incremental load, JOIN, and performance |
| `cdf-workflow` | CDF Workflows — task types, triggers, dependencies, and error handling |
| `cdf-naming-check` | Naming convention validation for CDF resources |
| `cog-vd-audit` | Full-project audit skill — scans all Toolkit YAML, SQL, and Python files and produces a structured `AUDIT_REPORT.md` |

Rules:

- `cdf-naming-conventions` — enforces CDF resource naming conventions across your project files

## Installation

### Claude Code

```bash
/plugin marketplace add cognitedata/cognite-ai-tooling-marketplace
/plugin install cog-vd-best-practices@cognite-ai-tooling-marketplace
```

### Cursor

Add the marketplace URL in Cursor Settings under **Plugins**, then install `cog-vd-best-practices`.

## Feedback

Share feedback or start a discussion in [Cognite Hub Groups](https://hub.cognite.com/groups/).

For security issues, see [SECURITY.md](./SECURITY.md).
