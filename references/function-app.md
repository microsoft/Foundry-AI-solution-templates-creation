# Azure Functions — Scaffold Pattern

This reference file defines the scaffold pattern for **serverless event-driven applications** using Azure Functions.

---

## Type-Specific Questions

| # | Question | Guidance |
|---|---|---|
| F1 | **Runtime stack?** | `Python` (default), `TypeScript/Node.js`, `C#/.NET`. |
| F2 | **Functions and their triggers?** | List each function: name, trigger type (HTTP, Timer, Queue, Blob, Event Grid, Cosmos DB change feed). |
| F3 | **Durable Functions needed?** | `No` (default), `Yes` (for long-running orchestrations, fan-out/fan-in). |
| F4 | **Azure Functions plan?** | `Consumption` (default, scale-to-zero, pay-per-execution), `Flex Consumption` (VNet, always-ready), `Premium` (no cold start, VNet). |
| F5 | **Input/output bindings?** | List bindings per function: e.g., "Queue trigger → Blob output", "HTTP trigger → Cosmos DB output". |
| F6 | **Shared state?** | `None` (default, stateless), `Cosmos DB`, `Table Storage`, `Redis`. |

---

## Project Folder Structure

### Python Runtime

```
<project-slug>/
├── function_app.py                 # Main function app entry point (v2 programming model)
├── host.json                       # Host-level configuration
├── local.settings.json             # Local development settings (git-ignored)
├── requirements.txt
│
├── functions/
│   ├── __init__.py
│   ├── <function-1>.py             # One file per function from F2
│   ├── <function-2>.py
│   └── shared/
│       ├── __init__.py
│       ├── config.py               # Shared configuration
│       ├── models.py               # Shared data models
│       └── utils.py                # Shared utilities
│
├── tests/
│   ├── __init__.py
│   ├── test_<function-1>.py
│   └── conftest.py
│
└── .funcignore                     # Files to exclude from deployment
```

### TypeScript Runtime

```
<project-slug>/
├── src/
│   ├── functions/
│   │   ├── <function-1>.ts
│   │   └── <function-2>.ts
│   └── shared/
│       ├── config.ts
│       ├── models.ts
│       └── utils.ts
├── host.json
├── local.settings.json
├── package.json
├── tsconfig.json
└── tests/
    └── <function-1>.test.ts
```

---

## Source File Patterns

### Python v2 Programming Model

#### `function_app.py`

```python
import azure.functions as func
from functions.<function_1> import bp as function_1_bp
from functions.<function_2> import bp as function_2_bp

app = func.FunctionApp()

# Register all function blueprints
app.register_functions(function_1_bp)
app.register_functions(function_2_bp)
```

#### HTTP Trigger Function

```python
# functions/<name>.py
import azure.functions as func
import json
import logging

bp = func.Blueprint()

@bp.function_name("<FunctionName>")
@bp.route(route="<endpoint>", methods=["POST"])
async def <function_name>(req: func.HttpRequest) -> func.HttpResponse:
    logging.info(f"Processing request to /<endpoint>")

    try:
        body = req.get_json()
        # Process the request
        result = await process(body)
        return func.HttpResponse(
            json.dumps(result),
            status_code=200,
            mimetype="application/json",
        )
    except ValueError as e:
        return func.HttpResponse(
            json.dumps({"error": str(e)}),
            status_code=400,
            mimetype="application/json",
        )
```

#### Timer Trigger Function

```python
@bp.function_name("<FunctionName>")
@bp.timer_trigger(schedule="0 */5 * * * *", arg_name="timer")  # Every 5 minutes
async def <function_name>(timer: func.TimerRequest) -> None:
    if timer.past_due:
        logging.warning("Timer is past due")
    # Execute scheduled work
    await process_scheduled_task()
```

#### Queue Trigger with Blob Output

```python
@bp.function_name("<FunctionName>")
@bp.queue_trigger(arg_name="msg", queue_name="<queue-name>", connection="AzureWebJobsStorage")
@bp.blob_output(arg_name="outputblob", path="<container>/{id}.json", connection="AzureWebJobsStorage")
async def <function_name>(msg: func.QueueMessage, outputblob: func.Out[str]) -> None:
    data = json.loads(msg.get_body().decode("utf-8"))
    result = await process(data)
    outputblob.set(json.dumps(result))
```

#### Durable Functions (if F3=Yes)

```python
# Orchestrator
import azure.functions as func
import azure.durable_functions as df

bp = func.Blueprint()

@bp.orchestration_trigger(context_name="context")
def orchestrator(context: df.DurableOrchestrationContext):
    """Fan-out/fan-in orchestration."""
    work_items = yield context.call_activity("GetWorkItems", None)

    tasks = [context.call_activity("ProcessItem", item) for item in work_items]
    results = yield context.task_all(tasks)

    summary = yield context.call_activity("Summarize", results)
    return summary

@bp.activity_trigger(input_name="item")
async def process_item(item: dict) -> dict:
    """Process a single work item."""
    return await do_processing(item)
```

### `host.json`

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensions": {
    "queues": {
      "batchSize": 16,
      "maxDequeueCount": 5
    }
  }
}
```

---

## Bicep Modules Required

- `monitoring.bicep` (always)
- `function-app.bicep` — Azure Functions app + hosting plan (from F4)
- `storage.bicep` — required for Functions runtime (AzureWebJobsStorage)
- `keyvault.bicep` — if secrets needed

If F5 includes queue/Service Bus bindings:
- `servicebus.bicep`

If F6 includes Cosmos DB:
- `cosmos.bicep`

### `infra/modules/function-app.bicep`

```bicep
param location string
param tags object
param functionAppName string
param appServicePlanName string
param storageAccountName string
param appInsightsConnectionString string
param planSku string = 'Y1'  // Consumption: Y1, Premium: EP1

resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: appServicePlanName
  location: location
  tags: tags
  sku: { name: planSku }
  kind: 'functionapp'
  properties: { reserved: true }  // Linux
}

resource functionApp 'Microsoft.Web/sites@2023-12-01' = {
  name: functionAppName
  location: location
  tags: tags
  kind: 'functionapp,linux'
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: plan.id
    siteConfig: {
      linuxFxVersion: 'Python|3.12'
      appSettings: [
        { name: 'AzureWebJobsStorage__accountName', value: storageAccountName }
        { name: 'FUNCTIONS_EXTENSION_VERSION', value: '~4' }
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'python' }
        { name: 'APPLICATIONINSIGHTS_CONNECTION_STRING', value: appInsightsConnectionString }
      ]
    }
  }
}

output functionAppName string = functionApp.name
output principalId string = functionApp.identity.principalId
```

---

## Type-Specific Quality Checklist

- [ ] All functions listed in F2 are implemented with correct trigger types
- [ ] `host.json` configures appropriate settings for the trigger types used
- [ ] Input/output bindings match F5 specification
- [ ] Durable Functions patterns are correct if F3=Yes
- [ ] `local.settings.json` is in `.gitignore`
- [ ] Functions handle errors gracefully and return appropriate HTTP status codes
- [ ] Timer functions check `past_due` and handle accordingly
- [ ] Queue functions handle poison messages (maxDequeueCount)
- [ ] Tests mock Azure Functions context and trigger objects
- [ ] `.funcignore` excludes test files, docs, and dev-only files
