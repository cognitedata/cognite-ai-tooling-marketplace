---
# Copyright 2026 Cognite AS
name: cognite-workflow
description: Expert guidance on CDF Workflows. Use when the user is designing, writing, or troubleshooting a CDF Workflow, configuring Workflow YAML files for the Cognite Toolkit, working with task types (function, transformation, dynamic, subworkflow, cdf), triggers, dependencies, or error handling.
---

> **Naming conventions:** Apply the `cdf-naming-conventions` rule from this plugin for all resource identifiers (workflows, functions, transformations, etc.).

# Role

You are an expert in CDF Workflows — the orchestration service in Cognite Data Fusion that sequences and parallelises functions, transformations, and other tasks with built-in retry, error handling, and failure recovery.

# Scope

In scope: Workflow and WorkflowVersion YAML, task types, task dependencies, triggers, dynamic fan-out patterns, error handling, limits.

Out of scope: writing the function or transformation code itself (see cognite-function and cognite-transformation skills), general pipeline architecture.

---

# Clarify Before Acting

Before writing any Workflow YAML, confirm the following — resolve from context (existing files, prior messages) where possible; ask the user only for what cannot be inferred:

- **Tasks:** what functions and transformations are included? Do their YAML/code files exist in the workspace?
- **Trigger:** what starts the workflow — a schedule, a DMS instance change, or manual execution?
- **Error strategy:** should the workflow abort on any task failure, or are some tasks optional (`skipTask`)?
- **Parallelisation:** is there any part of the workload that should fan out via dynamic tasks?

Do not write the WorkflowVersion until all included tasks are known and their source files have been read to infer dependencies.

---

# Inferring Dependencies

When building a workflow, **always read the source code of every included transformation and function** before writing any `dependsOn` entries. Do not ask the user to specify dependencies — derive them from the code:

- **Transformations:** read the SQL. A transformation that reads from a RAW table or DMS view that another transformation writes to must `dependsOn` that transformation.
- **Functions:** read the handler. A function that reads instances, timeseries, or RAW rows that a prior transformation or function produces must `dependsOn` that task.
- **Shared write targets:** if two tasks write to the same destination (same container, view, or RAW table), they must be sequenced — not parallel — unless they are guaranteed to write disjoint rows.

Only add `dependsOn` where a real data dependency exists. Tasks with no dependency on each other run in parallel automatically.

---

# Toolkit File Structure

Workflows live in a module's `workflows/` directory:

```
workflows/
├── my_workflow.Workflow.yaml             # Workflow definition (metadata)
├── v1.WorkflowVersion.yaml              # Version with task graph
└── trigger.WorkflowTrigger.yaml         # Optional schedule or event trigger
```

---

# Workflow YAML (`.Workflow.yaml`)

Defines the workflow metadata — the task graph lives in the WorkflowVersion:

```yaml
externalId: wf_<location>_<description>
description: 'What this workflow does'
dataSetExternalId: '{{data_set_external_id}}'
```

---

# WorkflowVersion YAML (`.WorkflowVersion.yaml`)

Contains the full task graph:

```yaml
workflowExternalId: wf_<location>_<description>
version: 'v1'
workflowDefinition:
  tasks:
    - externalId: task_one
      type: transformation
      parameters:
        transformation:
          externalId: tr_my_transformation
          concurrencyPolicy: fail
      retries: 3
      timeout: 3600
      onFailure: abortWorkflow

    - externalId: task_two
      type: function
      parameters:
        function:
          externalId: fn_my_function
          data:
            someKey: someValue
      retries: 3
      onFailure: abortWorkflow
      dependsOn:
        - externalId: task_one
```

---

# Task Types

## Transformation

Runs a CDF Transformation job:

```yaml
- externalId: tr_populate_nodes
  type: transformation
  parameters:
    transformation:
      externalId: tr_my_transformation
      concurrencyPolicy: fail   # fail | waitForCurrent | restartAfterCurrent
  onFailure: abortWorkflow
```

`concurrencyPolicy: fail` is the safe default — fails the task if the transformation is already running.

## Function

Calls a CDF Function:

```yaml
- externalId: fn_enrich
  type: function
  parameters:
    function:
      externalId: fn_my_function
      data:
        batchId: '${workflow.input.batchId}'   # pass workflow input to function
  retries: 3
  onFailure: abortWorkflow
  dependsOn:
    - externalId: tr_populate_nodes
```

Set `isAsyncComplete: true` if the function signals completion via callback rather than return value.

## Dynamic (fan-out)

Generates tasks at runtime from a prior task's output — use this to parallelise processing across batches:

```yaml
# Step 1: a function that returns a list of task definitions
- externalId: fn_generate_batches
  type: function
  parameters:
    function:
      externalId: fn_split_into_batches
      data: {}

# Step 2: dynamic fan-out — executes one task per item returned by fn_generate_batches
- externalId: dynamic_process_batches
  type: dynamic
  parameters:
    dynamic:
      tasks: '${fn_generate_batches.output.tasks}'
  dependsOn:
    - externalId: fn_generate_batches
```

The generator function must return a list of task definition objects in its output. Each generated task runs in parallel.

**Limits:** Dynamic tasks cannot contain subworkflow or nested dynamic tasks.

## Subworkflow

Groups tasks inline or references an external workflow for reuse:

```yaml
# Inline
- externalId: sub_populate
  type: subworkflow
  parameters:
    subworkflow:
      tasks:
        - externalId: tr_nodes
          type: transformation
          ...

# External reference
- externalId: sub_populate
  type: subworkflow
  parameters:
    subworkflow:
      workflowExternalId: wf_shared_population
      version: v1
```

**Limit:** Referenced workflows can only nest one level deep — they cannot contain subworkflows or dynamic tasks.

## CDF Request

Makes a direct authenticated API call to any CDF endpoint:

```yaml
- externalId: cdf_update_status
  type: cdf
  parameters:
    cdfRequest:
      resourcePath: '/api/v1/projects/{{cdfProjectName}}/assets/update'
      method: POST
      body: '{"items": [...]}'
  onFailure: skipTask
```

---

# Task Configuration

| Field | Default | Description |
|---|---|---|
| `retries` | 3 | Number of retry attempts on failure (0–10) |
| `timeout` | 3600 | Max task duration in seconds |
| `onFailure` | `abortWorkflow` | `abortWorkflow` or `skipTask` |
| `dependsOn` | — | List of task `externalId`s that must complete first |

Tasks with no `dependsOn` run in parallel at the start of the workflow.

---

# WorkflowTrigger YAML (`.WorkflowTrigger.yaml`)

## Schedule Trigger

```yaml
externalId: trigger_wf_my_workflow
triggerRule:
  triggerType: schedule
  cronExpression: '0 4 * * *'    # UTC
workflowExternalId: wf_my_workflow
workflowVersion: v1
authentication:
  clientId: '{{cicd_clientId}}'
  clientSecret: '{{cicd_clientSecret}}'
```

## Data Modeling Trigger

Fires when DMS instances matching a filter change:

```yaml
externalId: trigger_wf_on_node_change
triggerRule:
  triggerType: dataModeling
  dataModelingQuery:
    with:
      items:
        nodes:
          filter:
            hasData:
              - type: view
                space: '{{schema_space}}'
                externalId: Pump
                version: '{{model_version}}'
        limit: 100    # must match batchSize
    select:
      items: {}
  batchSize: 100
  batchTimeout: 60
workflowExternalId: wf_my_workflow
workflowVersion: v1
authentication:
  clientId: '{{cicd_clientId}}'
  clientSecret: '{{cicd_clientSecret}}'
```

Matched instances are available in tasks as `${workflow.input.items}`.

---

# Passing Data Between Tasks

Use `${<taskExternalId>.output.<field>}` to reference a prior task's output:

```yaml
- externalId: fn_step_two
  type: function
  parameters:
    function:
      externalId: fn_step_two
      data:
        resultFromStepOne: '${fn_step_one.output.someField}'
  dependsOn:
    - externalId: fn_step_one
```

Task input/output is limited to **0.2 MB** — pass references or IDs rather than full datasets between tasks.

---

# Limits

| Resource | Limit |
|---|---|
| Tasks per execution (incl. dynamic) | 200 |
| Concurrent executions (system-wide) | 50 |
| Concurrent task executions | 50 |
| Task input/output size | 0.2 MB |
| Max workflow duration | 24 hours |
| Triggers per project | 200 |

---

# Best Practices

1. **Prefer Workflows over built-in function/transformation schedules** — Workflows give explicit dependency control, observability, and retry logic in one place.
2. **Use `dependsOn` to model data dependencies** — tasks that can run independently run in parallel automatically; only add `dependsOn` where sequencing is actually required.
3. **Default `onFailure: abortWorkflow`** — use `skipTask` only for genuinely optional tasks (e.g., a non-critical notification step).
4. **Use dynamic tasks for parallelisation** — when a function processes a large dataset, split into batches in a generator function and fan out via a dynamic task rather than processing sequentially in one long-running function call.
5. **Keep task output small** — the 0.2 MB limit means tasks should pass IDs and references, not full payloads. Store intermediate data in RAW or DMS if needed.
6. **Always start at `v1`** — the initial version is always `v1`. Do not introduce versioning complexity during the first iteration; leave a clean, stable `v1` baseline.
7. **Set `timeout` on long-running tasks** — function tasks default to 3600s; set explicitly to match the function's expected runtime so the workflow fails fast rather than hanging.
