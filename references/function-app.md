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
| F7 | **Include AI inference functions?** | `No` (default). If `Yes`: functions that call Foundry for AI processing. Triggers U11 question. |
| F8 | **What AI function patterns?** | Only ask if F7=Yes. Options: `HTTP-triggered inference` (synchronous AI endpoint), `Queue-triggered batch` (async batch processing), `Timer-triggered enrichment` (scheduled AI processing). |

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

## AI Inference Functions (F7 = Yes, U11 = Yes)

**Important architectural note:** Azure Functions cannot host the MAF agent sidecar. The Foundry agent runs as a **separate Container App**, and Functions call it via REST. Read `references/foundry-agent-patterns.md` for the shared agent patterns.

### Additional Folder Structure

```
<project-slug>/
├── agents/
│   └── inference-agent/
│       ├── agent.yaml
│       ├── Dockerfile
│       ├── main.py
│       ├── requirements.txt
│       ├── schemas.py
│       └── skills/
│           └── inference-skill/
│               └── skill.md
├── functions/
│   └── shared/
│       └── foundry_client.py           # Foundry REST client for Functions
├── scripts/
│   └── register_agents.py
```

### Foundry Client for Functions

Functions call the Foundry agent via REST (not the two-mode dispatcher, since Functions don't run in Docker Compose locally):

```python
# functions/shared/foundry_client.py
"""Foundry Agent REST client for Azure Functions."""
import os
import json
import httpx
from azure.identity import DefaultAzureCredential


def get_foundry_token() -> str:
    credential = DefaultAzureCredential()
    token = credential.get_token("https://cognitiveservices.azure.com/.default")
    return token.token


async def call_foundry_agent(agent_name: str, payload: dict) -> dict:
    """Call a Foundry hosted agent via REST API."""
    endpoint = os.environ["AZURE_AI_PROJECT_ENDPOINT"]
    token = get_foundry_token()

    async with httpx.AsyncClient(timeout=120) as client:
        response = await client.post(
            f"{endpoint}/responses",
            json={**payload, "agent_reference": agent_name},
            headers={"Authorization": f"Bearer {token}"},
        )
        response.raise_for_status()
        return response.json()
```

### AI Function Patterns (based on F8)

#### HTTP-Triggered Inference

```python
# functions/ai_inference.py
"""HTTP-triggered AI inference function."""
import azure.functions as func
import json
from .shared.foundry_client import call_foundry_agent
import asyncio

bp = func.Blueprint()


@bp.function_name("AIInference")
@bp.route(route="inference", methods=["POST"])
async def ai_inference(req: func.HttpRequest) -> func.HttpResponse:
    try:
        body = req.get_json()
        result = await call_foundry_agent("inference-agent", body)
        return func.HttpResponse(
            json.dumps(result), status_code=200, mimetype="application/json",
        )
    except Exception as e:
        return func.HttpResponse(
            json.dumps({"error": str(e)}), status_code=500, mimetype="application/json",
        )
```

#### Queue-Triggered Batch Processing

```python
# functions/ai_batch.py
"""Queue-triggered batch AI processing."""
import azure.functions as func
import json
from .shared.foundry_client import call_foundry_agent

bp = func.Blueprint()


@bp.function_name("AIBatchProcess")
@bp.queue_trigger(arg_name="msg", queue_name="ai-batch-queue", connection="AzureWebJobsStorage")
@bp.blob_output(arg_name="result", path="ai-results/{id}.json", connection="AzureWebJobsStorage")
async def ai_batch(msg: func.QueueMessage, result: func.Out[str]) -> None:
    data = json.loads(msg.get_body().decode("utf-8"))
    ai_result = await call_foundry_agent("inference-agent", data)
    result.set(json.dumps(ai_result))
```

#### Timer-Triggered Enrichment

```python
# functions/ai_enrich.py
"""Timer-triggered AI enrichment — scheduled batch processing."""
import azure.functions as func
import logging
from .shared.foundry_client import call_foundry_agent

bp = func.Blueprint()


@bp.function_name("AIEnrichment")
@bp.timer_trigger(schedule="0 0 */6 * * *", arg_name="timer")  # Every 6 hours
async def ai_enrich(timer: func.TimerRequest) -> None:
    if timer.past_due:
        logging.warning("Timer is past due")
    # Fetch records to enrich, call agent, store results
    # Implementation depends on data source
    logging.info("AI enrichment job complete")
```

### AI Mode — Hand-Rolled (F7 = Yes, U11 = No)

If the user declines Foundry, use direct Azure OpenAI SDK inside function code:

```python
# functions/shared/ai_service.py
import os
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider


def get_ai_client() -> AzureOpenAI:
    token_provider = get_bearer_token_provider(
        DefaultAzureCredential(),
        "https://cognitiveservices.azure.com/.default",
    )
    return AzureOpenAI(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        azure_ad_token_provider=token_provider,
        api_version="2024-10-21",
    )


def process_with_ai(text: str) -> dict:
    client = get_ai_client()
    response = client.chat.completions.create(
        model=os.environ.get("AZURE_OPENAI_DEPLOYMENT_NAME", "gpt-4o"),
        messages=[
            {"role": "system", "content": "Process this input. Return JSON."},
            {"role": "user", "content": text},
        ],
        response_format={"type": "json_object"},
    )
    import json
    return json.loads(response.choices[0].message.content)
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

**If F7 = Yes and U11 = Yes (Foundry AI mode):**
- `ai-foundry.bicep` — Foundry account + project + model deployment
- `container-apps-env.bicep` + `container-app.bicep` — for the inference agent (runs as Container App, NOT as a Function)
- `container-registry.bicep` — for agent container image
- RBAC: `Cognitive Services OpenAI User` + `Azure AI User` on Foundry account (for both Function App and agent container)

**If F7 = Yes and U11 = No (Hand-rolled AI mode):**
- `ai-foundry.bicep` — Azure OpenAI model deployment only

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

**If F7 = Yes and U11 = Yes (Foundry AI mode) — also verify:**
- [ ] Inference agent runs as Container App (NOT inside Functions runtime)
- [ ] `foundry_client.py` uses `DefaultAzureCredential` for Foundry REST calls
- [ ] AI functions registered in `function_app.py` via blueprint
- [ ] Function App has `AZURE_AI_PROJECT_ENDPOINT` app setting in Bicep
- [ ] Run Foundry quality checklist from `references/foundry-agent-patterns.md`
