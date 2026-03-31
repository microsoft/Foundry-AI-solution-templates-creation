# API Backend — Scaffold Pattern

This reference file defines the scaffold pattern for **REST or GraphQL API backend** services deployed on Azure Container Apps.

---

## Type-Specific Questions

| # | Question | Guidance |
|---|---|---|
| A1 | **API style?** | `REST` (default), `GraphQL`. Drives router/resolver structure. |
| A2 | **Framework?** | `FastAPI` (default for Python), `Flask`, `Express` (TypeScript), `ASP.NET` (C#). |
| A3 | **Database?** | `Cosmos DB` (default), `PostgreSQL Flexible Server`, `MongoDB`, `None` (stateless). |
| A4 | **Key entities/resources?** | List the main data models (e.g., Users, Products, Orders). Drives schema + endpoint generation. |
| A5 | **Authentication method?** | `None` (default scaffold), `API Key`, `JWT/Entra ID`, `OAuth2`. |
| A6 | **Rate limiting?** | `None` (default), `IP-based`, `Token-based`. |
| A7 | **Pagination style?** | `Offset-based` (default), `Cursor-based`, `None`. |
| A8 | **API versioning?** | `URL path` (default, e.g., /v1/), `Header-based`, `None`. |
| A9 | **Does this API serve AI inference endpoints?** | `No` (default). If `Yes`: AI processing powered by Azure AI Foundry Agent Service. Triggers U11 question. |
| A10 | **What AI processing does the API perform?** | Only ask if A9=Yes. Options: `Classification`, `Summarization`, `Entity extraction`, `Content generation`, `Custom` (describe). Drives agent skill design. |

---

## Project Folder Structure

```
<project-slug>/
├── src/
│   ├── __init__.py
│   ├── main.py                     # App factory + lifespan
│   ├── config.py                   # pydantic-settings
│   ├── observability.py            # OTel setup
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── <entity>.py             # One router per entity from A4
│   │   └── health.py               # GET /health
│   ├── models/
│   │   ├── __init__.py
│   │   └── <entity>.py             # Pydantic schemas per entity
│   ├── services/
│   │   ├── __init__.py
│   │   └── <entity>_service.py     # Business logic per entity
│   ├── db/
│   │   ├── __init__.py
│   │   ├── client.py               # Database client (from A3)
│   │   └── repositories/
│   │       ├── __init__.py
│   │       └── <entity>_repo.py    # Data access per entity
│   └── middleware/
│       ├── __init__.py
│       ├── auth.py                 # Authentication (from A5)
│       ├── rate_limit.py           # Rate limiting (from A6)
│       └── error_handler.py        # Global error handling
│
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_<entity>.py            # Per-entity endpoint tests
    └── test_health.py
```

---

## Source File Patterns

### Router (per entity)

```python
from fastapi import APIRouter, Depends, HTTPException, Query
from ..models.<entity> import <Entity>Create, <Entity>Response, <Entity>Update
from ..services.<entity>_service import <Entity>Service

router = APIRouter(prefix="/<entities>", tags=["<Entities>"])

@router.get("", response_model=list[<Entity>Response])
async def list_items(skip: int = Query(0, ge=0), limit: int = Query(20, ge=1, le=100)):
    service = <Entity>Service()
    return await service.list(skip=skip, limit=limit)

@router.get("/{item_id}", response_model=<Entity>Response)
async def get_item(item_id: str):
    service = <Entity>Service()
    item = await service.get(item_id)
    if not item:
        raise HTTPException(status_code=404, detail="Not found")
    return item

@router.post("", response_model=<Entity>Response, status_code=201)
async def create_item(data: <Entity>Create):
    service = <Entity>Service()
    return await service.create(data)

@router.put("/{item_id}", response_model=<Entity>Response)
async def update_item(item_id: str, data: <Entity>Update):
    service = <Entity>Service()
    item = await service.update(item_id, data)
    if not item:
        raise HTTPException(status_code=404, detail="Not found")
    return item

@router.delete("/{item_id}", status_code=204)
async def delete_item(item_id: str):
    service = <Entity>Service()
    deleted = await service.delete(item_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Not found")
```

### Pydantic Models (per entity)

```python
from pydantic import BaseModel, Field
from datetime import datetime

class <Entity>Base(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    # Add entity-specific fields from A4

class <Entity>Create(<Entity>Base):
    pass

class <Entity>Update(BaseModel):
    name: str | None = None
    # All fields optional for partial updates

class <Entity>Response(<Entity>Base):
    id: str
    created_at: datetime
    updated_at: datetime
```

---

## AI Capability Layer (A9 = Yes, U11 = Yes)

When the API includes AI endpoints powered by Foundry Agent Service, add these files to the project. Read `references/foundry-agent-patterns.md` for the shared patterns referenced below.

### Additional Folder Structure

```
<project-slug>/
├── agents/
│   └── <ai-task>-agent/               # e.g., classifier-agent, summarizer-agent
│       ├── agent.yaml                  # Foundry Hosted Agent descriptor
│       ├── Dockerfile
│       ├── main.py                     # MAF entry point
│       ├── requirements.txt
│       ├── schemas.py                  # Structured output model
│       └── skills/
│           └── <ai-task>-skill/
│               └── skill.md            # Agent domain rules
├── src/
│   ├── agents/
│   │   ├── __init__.py
│   │   └── hosted_agents.py            # Two-mode dispatcher
│   └── routers/
│       └── ai.py                       # NEW: AI inference endpoints
├── scripts/
│   └── register_agents.py              # Foundry agent registration
```

### AI Router

```python
# src/routers/ai.py
"""AI inference endpoints — powered by Foundry Agent Service."""
from fastapi import APIRouter
from pydantic import BaseModel
from ..agents.hosted_agents import dispatch

router = APIRouter(prefix="/ai", tags=["AI"])


class AIRequest(BaseModel):
    text: str
    # Add fields based on A10 answer


@router.post("/process")
async def ai_process(request: AIRequest):
    """AI inference endpoint. Routes to Foundry agent for processing."""
    result = await dispatch("<ai-task>-agent", {"text": request.text})
    return result
```

Register the AI router in `src/main.py`:
```python
from .routers import ai
app.include_router(ai.router)
```

### Agent Customization by A10

| A10 Answer | Agent Name | Skill Focus | Schema Fields |
|---|---|---|---|
| Classification | `classifier-agent` | Categorize input text into predefined classes | `category`, `confidence_score`, `reasoning` |
| Summarization | `summarizer-agent` | Condense text while preserving key information | `summary`, `key_points`, `word_count` |
| Entity extraction | `extractor-agent` | Extract structured entities from unstructured text | `entities[]` (name, type, span), `entity_count` |
| Content generation | `generator-agent` | Generate content based on input parameters | `generated_content`, `tone`, `word_count` |
| Custom | `<custom>-agent` | Based on user description | Based on user description |

Use the agent.yaml, main.py, schemas.py, skill.md, Dockerfile, and requirements.txt patterns from `references/foundry-agent-patterns.md`. Customize the schema and skill based on the A10 answer.

### AI Mode — Hand-Rolled (A9 = Yes, U11 = No)

If the user declines Foundry, add direct Azure OpenAI SDK calls instead:

```python
# src/services/ai_service.py
"""AI processing service using direct Azure OpenAI SDK."""
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


async def process_text(text: str, task: str = "classify") -> dict:
    """Process text using Azure OpenAI directly."""
    client = get_ai_client()
    response = client.chat.completions.create(
        model=os.environ.get("AZURE_OPENAI_DEPLOYMENT_NAME", "gpt-4o"),
        messages=[
            {"role": "system", "content": f"You are a {task} assistant. Return JSON."},
            {"role": "user", "content": text},
        ],
        response_format={"type": "json_object"},
        temperature=0.3,
    )
    import json
    return json.loads(response.choices[0].message.content)
```

---

## Bicep Modules Required

- `container-apps-env.bicep` + `container-app.bicep` (always)
- `container-registry.bicep` (always)
- `monitoring.bicep` (always)
- `cosmos.bicep` — if A3 = Cosmos DB
- `keyvault.bicep` — if A5 requires secrets

**If A9 = Yes and U11 = Yes (Foundry AI mode):**
- `ai-foundry.bicep` — Foundry account + project + model deployment
- Additional `container-app.bicep` instance for the agent container
- RBAC: `Cognitive Services OpenAI User` + `Azure AI User` on Foundry account

**If A9 = Yes and U11 = No (Hand-rolled AI mode):**
- `ai-foundry.bicep` — Azure OpenAI model deployment only

---

## Type-Specific Quality Checklist

- [ ] All CRUD endpoints exist for each entity from A4
- [ ] Pydantic models validate input (min/max length, type constraints)
- [ ] Database client uses `DefaultAzureCredential` (no connection strings in code)
- [ ] Global error handler returns consistent error response format
- [ ] Health endpoint returns service name and version
- [ ] Pagination implemented per A7 answer
- [ ] API versioning implemented per A8 answer
- [ ] Auth middleware applied if A5 != None
- [ ] Rate limiting middleware applied if A6 != None
- [ ] Tests cover all endpoints with happy path and error cases

**If A9 = Yes and U11 = Yes (Foundry AI mode) — also verify:**
- [ ] AI router registered in main.py
- [ ] Agent skill matches A10 answer (classification, summarization, etc.)
- [ ] AI endpoint returns structured JSON from Foundry agent
- [ ] Run Foundry quality checklist from `references/foundry-agent-patterns.md`
