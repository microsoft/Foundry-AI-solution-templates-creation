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

## Bicep Modules Required

- `container-apps-env.bicep` + `container-app.bicep` (always)
- `container-registry.bicep` (always)
- `monitoring.bicep` (always)
- `cosmos.bicep` — if A3 = Cosmos DB
- `keyvault.bicep` — if A5 requires secrets

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
