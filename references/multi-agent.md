# Multi-Agent System — Scaffold Pattern

This reference file defines the scaffold pattern for **Microsoft Foundry Multi-Agent Solution Accelerators** using the Microsoft Agent Framework (MAF). It produces a system where multiple specialized AI agents work together — coordinated by an orchestrator — and deployed as Foundry Hosted Agents.

---

## Type-Specific Questions

Ask these in addition to the universal questions (U1–U10):

| # | Question | Guidance |
|---|---|---|
| T1 | **Number and names of specialist agents** (recommend 3–5) | Each agent must have a distinct, non-overlapping scope. Always include a Synthesis/Decision agent as the final stage. |
| T2 | **Orchestration topology** | Options: `parallel-then-synthesis` (default), `batched-parallel-sequential-synthesis`, `fully-sequential`. |
| T3 | **External data sources / MCP tools needed per agent** (optional) | Determines which agents need `MCPStreamableHTTPTool`. List tool name, URL, auth type. |
| T4 | **Azure OpenAI model name** | Default: `gpt-4o`. Options: `gpt-4o`, `gpt-4.1`, `gpt-5.4`. Model choice determines available regions. |
| T5 | **Azure regions** | Constrained by model availability. `gpt-5.4`: `eastus2`, `swedencentral`. `gpt-4o`: most regions. |
| T6 | **Deployment SKU** | `GlobalStandard` (default) or `DataZoneStandard` (data residency, limited regions). |
| T7 | **Human-in-the-loop required?** | Override capability, review step. Drives frontend decision panel. |
| T8 | **What is the primary input?** | e.g., document, form, data record, text. Drives upload form and agent input contract. |
| T9 | **What is the primary output / decision?** | e.g., approve/reject, risk score, summary. Drives Synthesis agent design. |
| T10 | **Target SLA / latency?** | e.g., <2 min per case. Drives parallelism, timeouts, ACA scaling. |

---

## Project Folder Structure

```
<project-slug>/
├── agents/
│   ├── <agent-1>/
│   │   ├── agent.yaml                  # Foundry Hosted Agent descriptor
│   │   ├── Dockerfile
│   │   ├── main.py                     # MAF entry point (from_agent_framework)
│   │   ├── requirements.txt
│   │   ├── schemas.py                  # Pydantic structured output model
│   │   └── skills/
│   │       └── <agent-1>-skill/
│   │           └── skill.md            # Domain rules, decision criteria, output contract
│   └── <agent-N>/                      # (repeat for each agent from T1)
│
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── e2e_test.py                     # End-to-end integration tests
│   └── app/
│       ├── __init__.py
│       ├── main.py                     # FastAPI app factory + lifespan
│       ├── config.py                   # pydantic-settings Settings class
│       ├── observability.py            # OTel + Azure Monitor setup
│       ├── agents/
│       │   ├── __init__.py
│       │   ├── <agent-N>.py            # Per-agent HTTP dispatcher (one file per agent)
│       │   ├── orchestrator.py         # Fan-out/fan-in coordinator
│       │   └── hosted_agents.py        # Two-mode dispatcher (local vs Foundry)
│       ├── models/
│       │   ├── __init__.py
│       │   └── schemas.py              # Shared API request/response schemas
│       ├── routers/
│       │   ├── __init__.py
│       │   ├── agents.py               # /agents/* endpoints
│       │   └── review.py               # /review/* endpoints (submit, status, result)
│       └── services/
│           ├── __init__.py
│           └── storage.py              # In-memory store (swap for DB in prod)
│
├── frontend/                           # Include if U8=yes
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── next.config.ts
│   ├── package.json
│   ├── tsconfig.json
│   ├── postcss.config.mjs
│   ├── components.json                 # shadcn/ui config
│   ├── app/
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── upload-form.tsx             # Input submission UI
│   │   ├── progress-tracker.tsx        # Pipeline phase / agent status
│   │   ├── decision-panel.tsx          # Final output + human override (if T7=yes)
│   │   ├── agent-details.tsx           # Per-agent expandable result cards
│   │   ├── confidence-bar.tsx          # Visual confidence indicator
│   │   ├── header.tsx
│   │   └── ui/                         # shadcn/ui primitives
│   └── lib/
│       ├── api.ts                      # Typed fetch client → backend
│       ├── types.ts                    # TypeScript interfaces mirroring Pydantic schemas
│       ├── sample-case.ts              # Demo data for local testing
│       └── utils.ts
│
├── scripts/
│   ├── register_agents.py              # Foundry agent registration + start (postprovision)
│   └── check_agents.py                 # Agent health check
│
└── docs/
    ├── api-reference.md                # All API endpoints documented
    └── extending.md                    # How to add agents, swap models, add tools
```

---

## Source File Generation Instructions

### Per-Agent Files

For **each agent** identified in T1, generate all of the following:

#### `agents/<name>/agent.yaml`

```yaml
name: <agent-name>           # kebab-case — must match name used in register_agents.py
description: >               # REQUIRED — 2-3 sentences: what it does, tools, structured output
  <One-paragraph description of what this agent does, what tools/data it uses,
  and what structured output it produces.>
runtime: agent-framework
version: "1.0.0"
resources:
  cpu: "1"       # use "0.5" for reasoning-only agents with no MCP tools
  memory: "2Gi"  # use "1Gi" for lightweight agents
env:
  - name: AZURE_AI_PROJECT_ENDPOINT
    secretRef: azure-ai-project-endpoint
  - name: AZURE_OPENAI_DEPLOYMENT_NAME
    value: <model-deployment-name>   # from T4
  - name: APPLICATION_INSIGHTS_CONNECTION_STRING
    secretRef: app-insights-connection-string
  # Add per-agent MCP URLs here if this agent uses external tools (from T3)
```

#### `agents/<name>/main.py`

```python
"""<AgentName> Hosted Agent — MAF entry point."""
import os
from agent_framework import SkillsProvider
# Only import MCPStreamableHTTPTool if this agent uses MCP (from T3)
from agent_framework.azure import AzureOpenAIResponsesClient
from azure.ai.agentserver.agentframework import from_agent_framework
from azure.identity import DefaultAzureCredential
from schemas import <OutputSchema>


def main() -> None:
    # 1. Normalize App Insights env var (adapter expects no-underscore variant)
    _ai_conn = os.environ.get("APPLICATION_INSIGHTS_CONNECTION_STRING")
    if _ai_conn:
        os.environ.setdefault("APPLICATIONINSIGHTS_CONNECTION_STRING", _ai_conn)

    # 2. Load domain skill rules from skills/<name>-skill/skill.md
    skills = SkillsProvider.from_directory("skills")

    # 3. Wire MCP tools (only if this agent uses external tools from T3)
    # tools = [MCPStreamableHTTPTool(server_url=os.environ["MCP_<TOOL>_URL"], ...)]

    # 4. Build Azure OpenAI client using managed identity (no API keys)
    client = AzureOpenAIResponsesClient(
        endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        model=os.environ.get("AZURE_OPENAI_DEPLOYMENT_NAME", "<model-name>"),
    )

    # 5. Start hosted agent server — structured output enforced via response_format
    from_agent_framework(
        client=client,
        skills=skills,
        # tools=tools,
        default_options={"response_format": <OutputSchema>},
    ).run()


if __name__ == "__main__":
    main()
```

#### `agents/<name>/schemas.py`

```python
from pydantic import BaseModel, Field
from typing import Literal


class <OutputSchema>(BaseModel):
    """Structured output for <AgentName>. Every field must have a Field(description=...)."""

    summary: str = Field(description="Human-readable summary of findings")
    confidence_score: float = Field(ge=0.0, le=1.0, description="Composite confidence 0-1")
    confidence_level: Literal["HIGH", "MEDIUM", "LOW"]
    # --- Domain-specific fields go here based on agent responsibility ---
    errors: list[str] = Field(default_factory=list, description="Processing errors if any")
```

#### `agents/<name>/skills/<name>-skill/skill.md`

```markdown
# <Agent Name> — Domain Skill

## Role
You are a specialized <domain> agent. Your sole responsibility is: <one sentence scope>.
You do NOT perform tasks outside this scope — other agents handle those.

## Input Contract
You receive: <describe input fields>

## Instructions
<Step-by-step reasoning process. Be explicit:>
1. <First thing the agent should check/do>
2. <Second step — include decision criteria, thresholds, tool call conditions>
3. <Edge cases and fallback behavior>

## Output Requirements
- Always return a JSON object matching the <OutputSchema> Pydantic model exactly
- Never return free-form prose as your final response
- If data is missing, set confidence_level to LOW and describe gaps in `errors`
- confidence_score must reflect actual certainty, not optimism
```

#### `agents/<name>/requirements.txt`

```
agent-framework>=0.1.0
azure-ai-agentserver>=0.1.0
azure-identity>=1.17.0
pydantic>=2.7.0
opentelemetry-api>=1.24.0
azure-monitor-opentelemetry-exporter>=1.0.0b26
```

#### `agents/<name>/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN adduser --disabled-password --gecos "" appuser
USER appuser

EXPOSE 8000
CMD ["python", "main.py"]
```

---

### Backend Orchestrator

#### Orchestration Topologies (choose based on T2)

**Topology A — Parallel then Synthesis (default):**

```python
# backend/app/agents/orchestrator.py
import asyncio
from .hosted_agents import dispatch

async def orchestrate(data: dict) -> dict:
    """Fan-out to all specialist agents, then synthesize."""
    agent_names = [<list from T1, excluding synthesis agent>]

    # Phase 1: Parallel execution
    results = await asyncio.gather(
        *[dispatch(name, data) for name in agent_names],
        return_exceptions=True,
    )

    # Package results, handling any failures
    agent_results = {}
    for name, result in zip(agent_names, results):
        if isinstance(result, Exception):
            agent_results[name] = {"error": str(result)}
        else:
            agent_results[name] = result

    # Phase 2: Synthesis
    synthesis_input = {**data, "agent_results": agent_results}
    final = await dispatch("<synthesis-agent-name>", synthesis_input)
    return {"agent_results": agent_results, "synthesis": final}
```

**Topology B — Batched parallel then sequential then synthesis:**

```python
async def orchestrate(data: dict) -> dict:
    # Phase 1: Parallel batch
    phase1 = await asyncio.gather(
        dispatch("agent-1", data),
        dispatch("agent-2", data),
        return_exceptions=True,
    )
    # Phase 2: Sequential (depends on phase 1)
    phase2 = await dispatch("agent-3", {**data, "phase1": phase1})
    # Phase 3: Synthesis
    final = await dispatch("synthesis", {"phase1": phase1, "phase2": phase2})
    return {"phase1": phase1, "phase2": phase2, "synthesis": final}
```

**Topology C — Fully sequential:**

```python
async def orchestrate(data: dict) -> dict:
    r1 = await dispatch("agent-1", data)
    r2 = await dispatch("agent-2", {**data, "r1": r1})
    r3 = await dispatch("agent-3", {**data, "r1": r1, "r2": r2})
    final = await dispatch("synthesis", {"r1": r1, "r2": r2, "r3": r3})
    return {"r1": r1, "r2": r2, "r3": r3, "synthesis": final}
```

#### Two-Mode Dispatcher (`backend/app/agents/hosted_agents.py`)

```python
"""Two-mode dispatcher: direct HTTP (Docker Compose) vs Foundry routing (production)."""
import os
import httpx
from azure.identity.aio import DefaultAzureCredential

FOUNDRY_MODE = bool(os.environ.get("AZURE_AI_PROJECT_ENDPOINT"))

_credential = DefaultAzureCredential() if FOUNDRY_MODE else None


async def _get_token(scope: str) -> str:
    token = await _credential.get_token(scope)
    return token.token


async def dispatch(agent_name: str, payload: dict) -> dict:
    if FOUNDRY_MODE:
        endpoint = os.environ["AZURE_AI_PROJECT_ENDPOINT"]
        token = await _get_token("https://cognitiveservices.azure.com/.default")
        url = f"{endpoint}/responses"
        headers = {"Authorization": f"Bearer {token}"}
        body = {**payload, "agent_reference": agent_name}
    else:
        env_key = f"{agent_name.upper().replace('-', '_')}_URL"
        host = os.environ.get(env_key, f"http://{agent_name}:8000")
        url = f"{host}/responses"
        headers = {}
        body = payload

    async with httpx.AsyncClient(timeout=120) as client:
        response = await client.post(url, json=body, headers=headers)
        response.raise_for_status()
        return response.json()
```

#### Agent Registration Script (`scripts/register_agents.py`)

```python
"""Register all agents with Foundry — called in postprovision hook."""
import os
import yaml
from pathlib import Path
from azure.identity import DefaultAzureCredential
# Import Foundry SDK for agent registration

AGENTS_DIR = Path("agents")
PROJECT_ENDPOINT = os.environ["AZURE_AI_PROJECT_ENDPOINT"]
ACR_LOGIN_SERVER = os.environ["ACR_LOGIN_SERVER"]
IMAGE_TAG = os.environ["IMAGE_TAG"]


def register_all():
    credential = DefaultAzureCredential()

    for agent_dir in sorted(AGENTS_DIR.iterdir()):
        if not agent_dir.is_dir():
            continue
        agent_yaml = agent_dir / "agent.yaml"
        if not agent_yaml.exists():
            continue

        with open(agent_yaml) as f:
            agent_def = yaml.safe_load(f)

        agent_name = agent_def["name"]
        image = f"{ACR_LOGIN_SERVER}/{agent_name}:{IMAGE_TAG}"

        print(f"Registering agent: {agent_name} with image {image}")

        # Use Foundry SDK to:
        # 1. Create or update the agent version
        # 2. Pass description=agent_def["description"] to create_version()
        # 3. Start the agent
        # Registration code depends on the specific Foundry SDK version


if __name__ == "__main__":
    register_all()
```

---

### `spec.yaml` (Foundry Template Gallery)

Generate this file at the project root if the user intends to publish to the Foundry template gallery:

```yaml
type: apptemplate
name: <kebab-case-solution-name>
version: 1
display_name: <Human Readable Solution Name>
description: "<One sentence from U1>"
longDescription: "<3-5 sentences: agents, tech stack, orchestration pattern, deployment story>"
repository: https://github.com/<org>/<repo-name>
languages:
  - python
  - typescript     # remove if U8=no
author: <org>
models:
  - <model-name>   # from T4
services:
  - "Microsoft Foundry"
  - "Microsoft Agent Framework"
  - "Azure OpenAI"
  - "Azure Container Apps"
  - "Azure Container Registry"
  - "Azure Monitor"
  - "Azure Application Insights"
  # Add services from U7
templateType: SolutionTemplate
path: ./images
license: MIT
industry:
  - <Industry>     # Derived from U1/U2 context
tags:
  - multi-agent
  - agent-framework
  - foundry-hosted-agents
  - azure-container-apps
  - <domain-specific-tags>  # 3-5 tags derived from domain
regions:
  - <region-1>    # from T5
  - <region-2>
disclaimer: "With any AI solutions you create using these templates, you are responsible for assessing all associated risks, and for complying with all applicable laws and safety standards."
```

---

## Bicep Modules Required

Beyond the base modules (monitoring, container-registry, container-apps-env, container-app, role-assignments), multi-agent projects always need:

- `ai-foundry.bicep` — Foundry account + project + model deployment

Optional (based on T3 and U7):
- `cosmos.bicep` — if persistent storage selected
- `storage.bicep` — if document storage needed
- `keyvault.bicep` — if secrets beyond Foundry endpoint
- `servicebus.bicep` — if async agent triggering at scale

### Mandatory RBAC for Multi-Agent Projects

Every agent + backend container app managed identity must have **both** roles on the Foundry account scope:
- `Cognitive Services OpenAI User`: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`
- `Azure AI User`: `53ca9b11-8b9d-4b51-acae-26b3df39f6f0`

---

## Type-Specific Quality Checklist

- [ ] Every `agent.yaml` has a meaningful `description:` (2-3 sentences, not a placeholder)
- [ ] Every `main.py` uses `default_options={"response_format": Schema}` for structured output
- [ ] Every `schemas.py` has `confidence_score`, `confidence_level`, `summary`, and `errors` fields
- [ ] Every `skill.md` defines role, input contract, step-by-step instructions, and output requirements
- [ ] `register_agents.py` passes `description=agent_def["description"]` to `create_version()`
- [ ] For agents with MCP tools: `MCPStreamableHTTPTool` is wired in `main.py`
- [ ] `docker-compose.yml` does NOT set `AZURE_AI_PROJECT_ENDPOINT` on the backend (enables local mode)
- [ ] `hosted_agents.py` correctly switches between Foundry mode and direct HTTP mode
- [ ] `orchestrator.py` uses `asyncio.gather` for parallel agents (not sequential awaits)
- [ ] `config.py` uses pydantic-settings with `.env` file support
- [ ] Frontend `lib/types.ts` mirrors all Pydantic schemas (if frontend included)
- [ ] `progress-tracker.tsx` uses polling (not WebSocket) for simplicity (if frontend included)
- [ ] `decision-panel.tsx` includes human override UI if T7=yes (if frontend included)
