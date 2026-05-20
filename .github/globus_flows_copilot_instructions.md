# GitHub Copilot Instructions — Globus Flows Development
# Reference: https://docs.globus.org/api/flows/

## Role & Context

You are assisting a developer in authoring, deploying, and managing **Globus Flows** — a
Globus-hosted automation platform for orchestrating multi-step research data workflows. Flow
definitions are JSON documents that follow the Globus States Language. Runs are managed via the
Globus Python SDK, Globus CLI, or the Globus Flows REST API.

---

## Core Concepts

### Terminology
- **Action Provider**: An HTTP-accessible service that performs a single unit of work (e.g.,
  Globus Transfer, Globus Compute). Each exposes `/run`, `/status`, and `/cancel` endpoints.
- **Action**: A single, discrete invocation of an action provider. Has states: `ACTIVE`,
  `SUCCEEDED`, `FAILED`, `INACTIVE`.
- **Flow**: A JSON-defined orchestration of multiple action providers. Deployed to the Globus
  Flows service; itself implements the action provider interface (flows can call other flows).
- **Run**: A single execution of a deployed flow. Shares the action interface (status, cancel,
  release).
- **Input Schema**: A JSON Schema document that validates user input before a run starts.

---

## Flow Definition Structure

A valid flow definition is a JSON document with this top-level shape:

```json
{
  "Comment": "Human-readable description of the flow",
  "StartAt": "FirstStateName",
  "States": {
    "FirstStateName": { ... },
    "NextStateName":  { ... }
  }
}
```

- `StartAt` (required): Name of the first state to execute.
- `States` (required): Map of state names to state definitions.
- Every state must have either `"Next": "<StateName>"` or `"End": true`.

---

## State Types

### 1. Action State (most common)

Invokes an action provider and waits for completion.

```json
"TransferData": {
  "Type": "Action",
  "ActionUrl": "https://transfer.actions.globus.org/transfer",
  "ActionScope": "<optional-scope-string>",
  "Parameters": {
    "source_endpoint_id.$": "$.source_collection",
    "destination_endpoint_id.$": "$.destination_collection",
    "transfer_items": [
      {
        "source_path.$": "$.source_path",
        "destination_path.$": "$.destination_path",
        "recursive": true
      }
    ]
  },
  "ResultPath": "$.TransferResult",
  "WaitTime": 86400,
  "ExceptionOnActionFailure": true,
  "Catch": [
    {
      "ErrorEquals": ["ActionFailedException"],
      "ResultPath": "$.TransferError",
      "Next": "HandleTransferError"
    }
  ],
  "Next": "NextState"
}
```

**Key properties:**
| Property | Required | Description |
|---|---|---|
| `Type` | Yes | Must be `"Action"` |
| `ActionUrl` | Yes | Base URL of the action provider |
| `Parameters` or `InputPath` | Yes (one of) | Input to the action |
| `ResultPath` | Recommended | JSONPath where action output is stored |
| `WaitTime` | No (default: 300s) | Max seconds to wait for completion |
| `ExceptionOnActionFailure` | No (default: true) | Raise exception on FAILED status |
| `RunAs` | No (default: User) | Identity to run the action as |
| `ActionScope` | No | Override scope string for auth |
| `Catch` | No | Exception handler list |
| `Next` or `End` | Yes (one of) | Flow control |

### 2. ExpressionEval State

Computes derived values without calling an action. Use before `Choice` states or to build
final output.

```json
"ComputeLabel": {
  "Type": "ExpressionEval",
  "Parameters": {
    "label.=": "'Transfer-' + run_label",
    "count.$": "$.ItemCount"
  },
  "ResultPath": "$.Computed",
  "Next": "NextState"
}
```

### 3. Choice State

Branching logic based on flow state values.

```json
"CheckStatus": {
  "Type": "Choice",
  "Choices": [
    {
      "Variable": "$.TransferResult.status",
      "StringEquals": "SUCCEEDED",
      "Next": "PostProcess"
    },
    {
      "Variable": "$.ItemCount",
      "NumericGreaterThan": 0,
      "Next": "BatchProcess"
    }
  ],
  "Default": "HandleUnexpected"
}
```

### 4. Pass State

Passes or reshapes data without invoking any action.

```json
"SetDefaults": {
  "Type": "Pass",
  "Parameters": {
    "notify": true,
    "label.$": "$.run_label"
  },
  "ResultPath": "$.Config",
  "Next": "NextState"
}
```

### 5. Wait State

Pauses flow execution for a fixed duration.

```json
"WaitBeforeRetry": {
  "Type": "Wait",
  "Seconds": 300,
  "Next": "RetryTransfer"
}
```

### 6. Fail State

Forcibly terminates the run with an error.

```json
"AbortFlow": {
  "Type": "Fail",
  "Error": "TransferFailed",
  "Cause": "Source collection was unreachable"
}
```

---

## Parameter Types in Action States

Three ways to set values in `Parameters`:

| Syntax | Type | Example |
|---|---|---|
| `"key": value` | Constant | `"recursive": true` |
| `"key.$": "$.path"` | Reference (JSONPath) | `"source_path.$": "$.input_path"` |
| `"key.=": "expression"` | Expression | `"label.=": "'Job-' + job_id"` |

### JSONPath Conventions
- All references start with `$.` (root of flow state).
- Input to the flow is available at `$.` directly.
- `$._context` contains run metadata (run ID, flow ID, caller principal, etc.).

---

## Expressions

Expressions follow a Python-like syntax. Use in `"key.="` parameters.

```
# String concatenation
"full_path.=": "base_path + '/' + filename"

# Arithmetic
"total_size.=": "file_count * avg_size_mb"

# Conditional
"label.=": "custom_label if is_present('custom_label') else 'default-label'"
```

**Built-in functions:**
| Function | Description |
|---|---|
| `len(x)` | Length of string, object, or array |
| `pathsplit(path)` | Split path string → `[parent, basename]` |
| `is_present('key')` | Returns `true` if key exists in state |
| `getattr('key', default)` | Returns value or default if missing |

---

## Hosted Action Providers (Globus-managed)

| Service | Base URL | Type |
|---|---|---|
| Globus Transfer | `https://transfer.actions.globus.org/transfer` | Async |
| Globus Compute | `https://compute.actions.globus.org` | Async |
| Search Ingest | `https://actions.globus.org/search/ingest` | Async |
| Search Delete | `https://actions.globus.org/search/delete` | Async |
| Send Notification Email | `https://actions.globus.org/notification/notify` | Sync |
| Wait For User Selection | `https://actions.globus.org/weboption/wait_for_option` | Async |
| Expression Evaluation | `https://actions.globus.org/expression_eval` | Sync |
| Hello World (testing) | `https://actions.globus.org/hello_world` | Sync |
| DataCite Mint (DOI) | `https://actions.globus.org/datacite/mint` | Async |

To introspect an action provider's input schema:
```bash
curl 'https://actions.globus.org/hello_world' | jq '.input_schema'
```

---

## Input Schema

Define a JSON Schema document to validate user input before a run starts.

```json
{
  "type": "object",
  "required": ["source_collection", "destination_collection", "source_path"],
  "additionalProperties": false,
  "properties": {
    "source_collection": {
      "type": "string",
      "format": "uuid",
      "title": "Source Collection",
      "description": "UUID of the source Globus collection",
      "x-globus-collection": {}
    },
    "destination_collection": {
      "type": "string",
      "format": "uuid",
      "title": "Destination Collection",
      "x-globus-collection": {}
    },
    "source_path": {
      "type": "string",
      "title": "Source Path",
      "description": "Path on the source collection to transfer"
    },
    "notify_email": {
      "type": "string",
      "format": "email",
      "title": "Notification Email",
      "description": "Optional email for completion notification"
    }
  }
}
```

**Globus Web App hints (add to property):**
| Format / Key | Purpose |
|---|---|
| `"format": "globus-collection"` | Renders collection picker widget |
| `"format": "globus-principal"` | Renders identity/group picker widget |
| `"format": "globus-transfer-transfer#0.10"` | Renders transfer task body input |

---

## Protecting Secrets

Use `__Private_Parameters` to hide sensitive values from run introspection:

```json
"Parameters": {
  "api_key": "super-secret-value",
  "__Private_Parameters": ["api_key"]
}
```

---

## Exception Handling

```json
"Catch": [
  {
    "ErrorEquals": ["ActionUnableToRun"],
    "ResultPath": "$.Error",
    "Next": "NotifyFailure"
  },
  {
    "ErrorEquals": ["ActionFailedException"],
    "ResultPath": "$.Error",
    "Next": "RetryOrFail"
  },
  {
    "ErrorEquals": ["States.ALL"],
    "ResultPath": "$.Error",
    "Next": "GlobalErrorHandler"
  }
]
```

**Standard exception types:**
- `ActionUnableToRun` — Action could not be started
- `ActionFailedException` — Action ran but returned FAILED status
- `States.ALL` — Catch-all for any exception

---

## Globus Python SDK Usage

Install:
```bash
pip install globus-sdk
```

### Authenticate and Get a Flows Client

```python
import globus_sdk

# Use NativeAppAuthClient for interactive login
CLIENT_ID = "<your-native-app-client-id>"
client = globus_sdk.NativeAppAuthClient(CLIENT_ID)
client.oauth2_start_flow(
    requested_scopes=[globus_sdk.FlowsClient.scopes.all]
)
authorize_url = client.oauth2_get_authorize_url()
print(f"Please go to this URL and login: {authorize_url}")
auth_code = input("Enter the auth code: ").strip()
token_response = client.oauth2_exchange_code_for_tokens(auth_code)
flows_token = token_response.by_resource_server["flows.globus.org"]

authorizer = globus_sdk.AccessTokenAuthorizer(flows_token["access_token"])
flows_client = globus_sdk.FlowsClient(authorizer=authorizer)
```

### Deploy a Flow

```python
import json

with open("flow_definition.json") as f:
    flow_def = json.load(f)

with open("input_schema.json") as f:
    input_schema = json.load(f)

response = flows_client.create_flow(
    title="My HPC Data Transfer Flow",
    definition=flow_def,
    input_schema=input_schema,
    description="Transfers data from HPC scratch to archive and indexes it",
)
flow_id = response["id"]
print(f"Flow deployed: {flow_id}")
```

### Start a Run

```python
run_input = {
    "source_collection": "<source-collection-uuid>",
    "destination_collection": "<dest-collection-uuid>",
    "source_path": "/scratch/project/results/",
    "destination_path": "/archive/project/results/",
}

run = flows_client.run_flow(
    flow_id=flow_id,
    flow_scope=response["globus_auth_scope"],
    body=run_input,
    label="Run-2026-05-21",
)
run_id = run["run_id"]
print(f"Run started: {run_id}")
```

### Monitor a Run

```python
import time

while True:
    status = flows_client.get_run(run_id)
    print(f"Status: {status['status']}")
    if status["status"] in ("SUCCEEDED", "FAILED", "CANCELLED"):
        break
    time.sleep(30)
```

### List and Manage Flows

```python
# List all flows owned by the user
for flow in flows_client.list_flows(filter_role="flow_owner"):
    print(flow["id"], flow["title"])

# Update a flow definition
flows_client.update_flow(flow_id, definition=new_flow_def)

# Delete a flow
flows_client.delete_flow(flow_id)
```

---

## Globus CLI Usage

```bash
# Install
pip install globus-cli

# Login
globus login

# Validate a flow definition (before deploying)
globus flows validate flow_definition.json --input-schema input_schema.json

# Deploy a flow
globus flows create "My Flow Title" flow_definition.json   --input-schema input_schema.json   --description "Automates HPC data archival"

# List flows
globus flows list

# Start a run
globus flows run start <flow-id> --input run_input.json --label "Run-2026-05-21"

# Monitor a run
globus flows run show <run-id>

# List runs
globus flows run list --filter-flow-id <flow-id>

# Cancel a run
globus flows run cancel <run-id>
```

---

## Development Best Practices

### Flow Authoring
1. **Always include `ResultPath`** on Action states — without it, action output overwrites the
   entire flow state.
2. **Set `WaitTime` explicitly** for long-running actions (transfers, compute jobs). Default is
   only 300 seconds.
3. **Use `ExpressionEval` states** before `Choice` states when the branching value must be
   derived from multiple state fields.
4. **Prefer Guest Collections** over mapped collections to avoid mid-run consent prompts.
5. **Validate before deploying**: `globus flows validate` or the Flows IDE catches structural
   errors before they cause failed runs.
6. **Use `__Private_Parameters`** for any credentials or API keys embedded in flow definitions.

### State Design
- Give states descriptive names (`TransferToArchive`, not `State1`).
- Add a `Comment` field to complex states for documentation.
- Design `Catch` handlers for every Action state that interacts with external resources.
- Use a final `Pass` or `ExpressionEval` state to shape the output returned from the flow.

### Modular Flows
- Flows implement the action provider interface — use sub-flows to compose complex pipelines.
- Reference a deployed flow by its URL: `https://flows.globus.org/flows/<flow-id>`.

### Testing
- Use the `Hello World` action provider (`https://actions.globus.org/hello_world`) for
  structure testing without real side effects.
- Use the Flows IDE (https://flows.globus.org/ide) for real-time diagram visualization.

---

## $._context Object

Available in all flow states as `$._context`:

| Field | Description |
|---|---|
| `$._context.run_id` | UUID of the current run |
| `$._context.flow_id` | UUID of the flow |
| `$._context.flow_title` | Title of the flow |
| `$._context.caller` | Globus Auth identity of the run initiator |
| `$._context.run_owner` | Same as caller for user-initiated runs |

Use in Parameters:
```json
"run_id.$": "$._context.run_id"
```

---

## File Layout (Recommended Project Structure)

```
globus-flows/
├── flows/
│   ├── transfer_and_index/
│   │   ├── definition.json       # Flow definition
│   │   ├── input_schema.json     # Input schema
│   │   └── README.md
│   └── compute_and_transfer/
│       ├── definition.json
│       └── input_schema.json
├── scripts/
│   ├── deploy.py                 # Deploy/update flows via SDK
│   ├── run.py                    # Start runs programmatically
│   └── monitor.py                # Poll and log run status
├── tests/
│   └── validate_all.sh           # Runs `globus flows validate` on all definitions
└── requirements.txt              # globus-sdk, globus-cli
```

---

## References

- Globus Flows Docs: https://docs.globus.org/api/flows/
- Authoring Flows: https://docs.globus.org/api/flows/authoring-flows/
- Hosted Action Providers: https://docs.globus.org/api/flows/hosted-action-providers/
- Input Schema Guide: https://docs.globus.org/api/flows/input-schema/
- Example Flows: https://docs.globus.org/api/flows/examples/
- Flows IDE: https://flows.globus.org/ide
- Globus Python SDK: https://globus-sdk-python.readthedocs.io/
- Globus CLI Flows Reference: https://docs.globus.org/cli/reference/flows/
- API Specification: https://docs.globus.org/api/flows/reference/
