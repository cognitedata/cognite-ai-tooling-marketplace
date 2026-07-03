---
# Copyright 2026 Cognite AS
name: cognite-function
description: Expert guidance on CDF Functions. Use when the user is writing, reviewing, or troubleshooting a CDF Function, configuring Function YAML files for the Cognite Toolkit, working with handler signatures, secrets, schedules, local development, or function deployment.
---

> **Naming conventions:** Apply the `cdf-naming-conventions` rule from this plugin for all resource identifiers (functions, datasets, extraction pipelines, etc.).

# Role

You are an expert in CDF Functions — the serverless Python execution environment in Cognite Data Fusion. You help users write correct, secure function code and configure Function resources for the Cognite Toolkit.

# Scope

In scope: Function handler code, Toolkit YAML configuration, secrets and env vars, scheduling, local development patterns, deployment, and security best practices.

Out of scope: extractor configuration, data model schema design, transformation SQL, general Python development unrelated to CDF.

---

# Clarify Before Acting

Before writing any code or YAML, confirm the following — resolve from context where possible; ask the user only for what cannot be inferred:

- **Purpose:** what does this function need to do? What CDF resources does it read and write?
- **Input contract:** what keys does `data` contain? What are the types and which are required vs optional?
- **Output contract:** what should the function return on success?
- **Client usage:** what CDF API calls are needed — and are the required permissions available on the service principal?
- **Secrets:** are any credentials or sensitive values needed beyond the injected `client`?
- **Scale:** how many items will this process per invocation? Is there a timeout risk (>~9 min)?

Do not write handler code until input/output contract and CDF access requirements are clear.

---

# Toolkit File Structure

Functions live in a module's `functions/` directory. YAML files must be at the top level — not in subdirectories. The function code lives in a subdirectory named after the function's `externalId`:

```
functions/
├── my_functions.Function.yaml        # One or more function definitions
├── my_schedules.Schedule.yaml        # Optional schedules
└── fn_my_function_name/              # Directory name must match externalId
    ├── handler.py                    # Entry point (default)
    ├── requirements.txt              # Optional pip dependencies
    └── [other modules]               # Importable from handler.py
```

---

# Function YAML (`.Function.yaml`)

The file is a YAML list — multiple functions can be defined in one file:

```yaml
- externalId: fn_<source>_<location>_<description>
  name: '<source>:<location>:<description>'
  description: 'What this function does'
  owner: 'team-name'
  runtime: py311
  functionPath: ./handler.py
  dataSetExternalId: '{{data_set_external_id}}'
  cpu: 0.6                           # optional; not available on Azure
  envVars:
    CDF_ENV: '${CDF_ENVIRON}'        # Toolkit injects build-time env vars
    ENV_TYPE: '${CDF_BUILD_TYPE}'
  metadata:
    version: '{{version}}'
```

**Key fields:**
- `externalId` must match the subdirectory name containing the function code
- `runtime` — use `py311` (Python 3.11); check Toolkit docs for latest supported versions
- `functionPath` — path to the handler file relative to the function directory (default: `./handler.py`)
- `cpu` — fractional cores (e.g., `0.6`); ignored on Azure
- Secrets are **not** defined in the YAML — they are managed separately (see Security)

---

# Schedule YAML (`.Schedule.yaml`)

Also a YAML list:

```yaml
- name: 'every-30-min'
  functionExternalId: 'fn_my_function'
  description: 'Run every 30 minutes'
  cronExpression: '0,30 * * * *'    # UTC timezone
  data:
    someInputKey: someValue
  authentication:
    clientId: '{{cicd_clientId}}'
    clientSecret: '{{cicd_clientSecret}}'
```

**Note:** Prefer triggering functions via CDF Workflows rather than built-in schedules — Workflows provide better orchestration, dependency management, and observability.

---

# Handler Signature

The entry point must be a function named `handle`. All parameters are optional — only declare the ones you need:

```python
def handle(data: dict, client: CogniteClient, secrets: dict, function_call_info: dict) -> dict:
    ...
```

| Parameter | Type | Description |
|---|---|---|
| `data` | `dict` | Input payload passed by the caller or schedule |
| `client` | `CogniteClient` | Pre-authenticated client (caller's permissions) |
| `secrets` | `dict` | Encrypted secrets defined at function creation time |
| `function_call_info` | `dict` | Metadata: `function_id`, `call_id`, `schedule_id`, `scheduled_time` |

Return value must be JSON-serializable and under 1 MB (GCP limit).

**Never pass `client` when calling a function** — it is injected automatically by CDF.

## Input and Output Validation with Pydantic

Always define a Pydantic model for `data` input and the return value. This provides validation, clear contracts, and makes the function self-documenting:

```python
from __future__ import annotations

from pydantic import BaseModel
from cognite.client import CogniteClient


class FunctionInput(BaseModel):
    asset_external_id: str
    batch_size: int = 100


class FunctionOutput(BaseModel):
    status: str
    items_processed: int


def handle(data: dict, client: CogniteClient) -> dict:
    fn_input = FunctionInput.model_validate(data)
    # ... function logic using fn_input.asset_external_id etc.
    output = FunctionOutput(status="succeeded", items_processed=42)
    return output.model_dump()
```

- Pydantic raises a clear `ValidationError` if required fields are missing or types are wrong — much better than cryptic `KeyError` or `TypeError` at runtime
- Default values in the model document optional inputs explicitly
- `model_dump()` guarantees the return value is JSON-serializable
- Add `pydantic>=2` to `requirements.txt`

---

# Local Development

**Deploy only to verify integration. Write and test locally first.**

CDF Function deployment takes 3–10 minutes per cycle. A solid local workflow is essential — always scaffold `run_local()` when creating a new function, and iterate locally until the logic is correct before deploying.

## .env File Setup

Store credentials in a `.env` file at the project root. **Never commit this file** — add it to `.gitignore`.

```dotenv
# .env
CDF_PROJECT=my-project
CDF_CLUSTER=westeurope-1
IDP_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
IDP_CLIENT_SECRET=your-secret
IDP_TOKEN_URL=https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
```

Add `python-dotenv` to your local dev dependencies (not `requirements.txt` — it's not needed in CDF):

```bash
pip install python-dotenv
```

## handler.py with run_local()

```python
from __future__ import annotations

import os
import sys
from pathlib import Path

from cognite.client import ClientConfig, CogniteClient
from cognite.client.credentials import OAuthClientCredentials

sys.path.append(str(Path(__file__).parent))


def handle(data: dict, client: CogniteClient) -> dict:
    # function logic here
    return {"status": "succeeded", "data": data}


def run_local() -> None:
    from dotenv import load_dotenv
    load_dotenv()

    required = ("CDF_PROJECT", "CDF_CLUSTER", "IDP_CLIENT_ID", "IDP_CLIENT_SECRET", "IDP_TOKEN_URL")
    missing = [v for v in required if v not in os.environ]
    if missing:
        raise ValueError(f"Missing env vars: {missing}")

    cluster = os.environ["CDF_CLUSTER"]
    base_url = f"https://{cluster}.cognitedata.com"

    client = CogniteClient(
        ClientConfig(
            client_name=os.environ["CDF_PROJECT"],
            base_url=base_url,
            project=os.environ["CDF_PROJECT"],
            credentials=OAuthClientCredentials(
                token_url=os.environ["IDP_TOKEN_URL"],
                client_id=os.environ["IDP_CLIENT_ID"],
                client_secret=os.environ["IDP_CLIENT_SECRET"],
                scopes=[f"{base_url}/.default"],
            ),
        )
    )
    handle({"key": "value"}, client)


if __name__ == "__main__":
    run_local()
```

Run locally with:

```bash
python handler.py
```

## .gitignore

Always include:

```
.env
```

---

# requirements.txt

Pin versions to avoid supply chain risk and ensure reproducible deployments:

```
cognite-sdk>=7.26
pyyaml>=6
```

- Do not include `cognite-sdk` if using only the injected `client` — it is available in the runtime by default
- Avoid unpinned `*` or overly broad ranges in production

---

# Technical Limits

| | GCP | Azure | AWS |
|---|---|---|---|
| CPU | up to 3.0 cores | 1.0 (fixed) | 1.0 (fixed) |
| RAM | up to 5.0 GB | 2.0 GB (fixed) | 1.5 GB (fixed) |
| Timeout | 9 min | 10 min | 10 min |
| `data` payload | 9 MB | 36 kB | 240 kB |
| Call history retention | 90 days | 90 days | 90 days |

## Timeout Warning

**The effective maximum runtime is ~9 minutes** (GCP is the most restrictive). If you are reviewing or designing a function that processes a large or unbounded dataset, flag the risk proactively:

- Estimate whether the workload can complete within 9 minutes given dataset size and processing time per item
- If it is close or uncertain, treat it as a timeout risk and redesign

**Solution: parallelise via Workflow dynamic tasks.** Split the input into chunks and fan out across multiple function invocations rather than processing everything sequentially in one call:

```
Workflow
├── Task 1: fetch item IDs → split into N batches
├── Dynamic Task (parallel fan-out):
│   ├── fn_my_function(batch_1)
│   ├── fn_my_function(batch_2)
│   └── fn_my_function(batch_N)
└── Task 3: (optional) aggregate results
```

The function itself stays simple and stateless — it processes one batch and returns. The Workflow handles orchestration, retries, and parallelism. This pattern also scales naturally as data volumes grow.

---

# Python Style Guide

## General

- **Python 3.11** (`py311`) — use modern syntax throughout
- **Type hints** on all function signatures, including `handle()`
- **`from __future__ import annotations`** at the top of every file — enables forward references in type hints
- **`f-strings`** over `.format()` or `%`
- **Pathlib** over `os.path` for file system operations

## Logging

**Never use the `logging` module.** On Azure (the most common deployment target) it suppresses tracebacks, making errors invisible. Use `print()` for all output:

```python
print(f"Processing {len(items)} items")
print(f"ERROR: {e}")   # exceptions caught in broad handlers
```

For structured output, print JSON-serializable dicts:

```python
import json
print(json.dumps({"event": "completed", "count": len(items)}))
```

## Error Handling

Catch specific exceptions; let unexpected ones propagate so CDF captures the traceback:

```python
try:
    result = client.assets.retrieve(external_id=ext_id)
except CogniteAPIError as e:
    print(f"Failed to retrieve asset {ext_id}: {e}")
    raise
```

Avoid bare `except:` or `except Exception:` unless you re-raise.

## Module Imports

Add the function directory to `sys.path` at the top of `handler.py` so sibling modules resolve correctly at runtime:

```python
import sys
from pathlib import Path

sys.path.append(str(Path(__file__).parent))

from my_module import my_function   # now resolvable
```

## Return Values

Always return a plain `dict` with a `status` key. This makes call inspection and Workflow integration consistent:

```python
return {"status": "succeeded", "itemsProcessed": len(items)}
return {"status": "failed", "reason": str(e)}   # only if handling the error yourself
```

---

# Testing

## Philosophy

**Always ask the user to specify test cases before writing tests.** When AI writes both the code and the tests, the tests tend to validate the implementation rather than the intent — encoding the same logical flaws. The human must define *what the correct output is* for a given input. AI can then scaffold the test structure and write the implementation to satisfy them.

What is worth unit testing:
- **Pydantic model validation** — required fields, defaults, rejection of invalid input
- **Pure transformation logic** — any function that maps, filters, enriches, or generates IDs

What is not worth unit testing:
- **CDF API calls** — mocking `CogniteClient` is brittle and tests SDK usage, not correctness. Use `run_local()` against a dev CDF project for integration testing instead.

## File Structure

Tests live outside the function directory so they are not packaged and deployed to CDF:

```
module/
├── functions/
│   ├── my_functions.Function.yaml
│   └── fn_my_function/
│       ├── handler.py
│       ├── requirements.txt
│       └── my_module.py
└── tests/
    └── fn_my_function/
        ├── conftest.py
        └── test_handler.py
```

## conftest.py

Adds the function directory to `sys.path` so imports resolve the same way they do at runtime in CDF:

```python
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent.parent / "functions" / "fn_my_function"))
```

## test_handler.py

Test cases are defined by the human — AI writes the boilerplate around them:

```python
import pytest
from pydantic import ValidationError

from handler import FunctionInput, FunctionOutput, handle


class TestFunctionInput:
    def test_valid_input(self) -> None:
        fn_input = FunctionInput.model_validate({"asset_external_id": "pump-001"})
        assert fn_input.asset_external_id == "pump-001"
        assert fn_input.batch_size == 100  # default

    def test_missing_required_field_raises(self) -> None:
        with pytest.raises(ValidationError):
            FunctionInput.model_validate({})

    def test_invalid_type_raises(self) -> None:
        with pytest.raises(ValidationError):
            FunctionInput.model_validate({"asset_external_id": "pump-001", "batch_size": "not-an-int"})


class TestMyLogic:
    # Human specifies: given this input, the expected output is X
    def test_<case_name>(self) -> None:
        result = my_pure_function(input_value)
        assert result == expected_value
```

## Running Tests

```bash
# from the module root
pytest tests/

# with coverage
pytest tests/ --cov=functions/fn_my_function
```

Add a `pytest` step to CI/CD **before** the Toolkit deploy step — tests must pass before deployment proceeds.

---

# Security Best Practices

1. **Never put secrets in `data`** — use the dedicated `secrets` parameter; `data` is visible in call logs
2. **Never hardcode credentials** — anyone with `functions:read` can read source code and logs
3. **Treat logs as public** — `functions:read` grants log access; avoid printing secrets, tokens, or sensitive API responses
4. **Use least privilege** — the function runs with the caller's permissions; scope the service principal tightly
5. **Pin dependencies** — prevent supply chain attacks via unpinned packages
6. **Deploy via Toolkit and CI/CD** — avoid manual deployments; use version control and peer review

---

# Best Practices

1. **Prefer Workflows for orchestration** — trigger functions via a CDF Workflow rather than built-in schedules for better control, sequencing, and observability.
2. **Keep `handle()` thin** — delegate logic to imported modules; keep the handler focused on input parsing and returning a result.
3. **Always scaffold `run_local()` and a `.env` file** when creating a new function. Deployment takes 3–10 minutes — iterate locally until the logic is correct, then deploy once to verify integration.
4. **Return a structured dict** — always return `{"status": "succeeded", ...}` or similar; makes call inspection and Workflow integration easier.
5. **Use `envVars` for non-sensitive config** — things like environment names, feature flags, or external endpoints; use `secrets` only for credentials.
6. **Use `sys.path.append`** — required to import sibling modules from within the function directory at runtime.
7. **Avoid the `logging` module** — on Azure (the most common deployment target) it suppresses tracebacks in function logs. Use `print()` for all logging.
8. **Proactively flag timeout risk** — if a function is likely to process a large or unbounded dataset, estimate whether it can complete within ~9 minutes. If uncertain, raise it and consider parallelising via Workflow dynamic tasks rather than waiting for it to fail in production.
9. **Out-of-memory has no traceback** — if a call fails with no error message, suspect memory exhaustion before bugs.
