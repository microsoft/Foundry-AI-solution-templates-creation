# Foundry Agent Patterns — Shared Reference

Use these patterns when U11 = Yes (Azure AI Foundry Agent Service selected) for any AI-capable project type. These patterns produce a single Foundry-hosted agent (or small set of task-specific agents) integrated into an existing project.

**Do NOT load this file for Multi-Agent projects** — `references/multi-agent.md` has its own complete multi-agent orchestration patterns with fan-out/fan-in topologies, multiple specialist agents, and a synthesis agent.

---

## When This File Applies

| Project Type | Condition | Agent Purpose |
|---|---|---|
| RAG Chatbot | Always (U11=Yes default) | Retrieval-grounded Q&A agent |
| API Backend | A9=Yes, U11=Yes | AI inference endpoint agent |
| Full-Stack Web App | W10=Yes, U11=Yes | AI feature agent (chat, summarization, etc.) |
| Data Pipeline | D9=Yes, U11=Yes | AI data enrichment agent |
| Azure Functions | F7=Yes, U11=Yes | AI inference agent (runs as Container App sidecar) |
| Event-Driven | E9=Yes, U11=Yes | AI event processor agent |

---

## Agent Descriptor (`agent.yaml`)

Every Foundry agent needs a declarative YAML descriptor:

```yaml
name: <agent-name>           # kebab-case — must match name in register_agents.py
description: >               # REQUIRED — 2-3 sentences: what it does and what it produces
  <One-paragraph description of what this agent does, what inputs it processes,
  and what structured output it produces.>
runtime: agent-framework
version: "1.0.0"
resources:
  cpu: "1"       # use "0.5" for lightweight reasoning-only agents
  memory: "2Gi"  # use "1Gi" for lightweight agents
env:
  - name: AZURE_AI_PROJECT_ENDPOINT
    secretRef: azure-ai-project-endpoint
  - name: AZURE_OPENAI_DEPLOYMENT_NAME
    value: <model-deployment-name>      # from U11 context or type-specific question
  - name: APPLICATION_INSIGHTS_CONNECTION_STRING
    secretRef: app-insights-connection-string
  # Add additional env vars as needed by the specific agent
```

The `agent.yaml` separates declarative metadata from code, enabling Foundry to manage agent lifecycle independently.

---

## MAF Entry Point (`main.py`)

```python
"""<AgentName> Hosted Agent — MAF entry point."""
import os
from agent_framework import SkillsProvider
from agent_framework.azure import AzureOpenAIResponsesClient
from azure.ai.agentserver.agentframework import from_agent_framework
from azure.identity import DefaultAzureCredential
from schemas import <OutputSchema>


def main() -> None:
    # 1. Normalize App Insights env var (MAF expects no-underscore variant)
    _ai_conn = os.environ.get("APPLICATION_INSIGHTS_CONNECTION_STRING")
    if _ai_conn:
        os.environ.setdefault("APPLICATIONINSIGHTS_CONNECTION_STRING", _ai_conn)

    # 2. Load domain skill rules from skills/<name>-skill/skill.md
    skills = SkillsProvider.from_directory("skills")

    # 3. Optional: Wire MCP tools for external data access
    # from agent_framework import MCPStreamableHTTPTool
    # tools = [MCPStreamableHTTPTool(server_url=os.environ["MCP_TOOL_URL"])]

    # 4. Build Azure OpenAI client using managed identity (no API keys)
    client = AzureOpenAIResponsesClient(
        endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        model=os.environ.get("AZURE_OPENAI_DEPLOYMENT_NAME", "gpt-4o"),
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

Key behaviors:
- `SkillsProvider` loads domain rules from skill.md files (not hardcoded prompts)
- `from_agent_framework()` wraps the agent in Foundry hosting runtime — no manual HTTP setup
- `response_format=<OutputSchema>` enforces structured output at the agent level
- `DefaultAzureCredential` provides managed identity auth — no API keys in code

---

## Structured Output Schema (`schemas.py`)

Every Foundry agent must produce structured output via a Pydantic model:

```python
from pydantic import BaseModel, Field
from typing import Literal


class <OutputSchema>(BaseModel):
    """Structured output for <AgentName>. Every field must have a Field(description=...)."""

    summary: str = Field(description="Human-readable summary of findings")
    confidence_score: float = Field(ge=0.0, le=1.0, description="Composite confidence 0-1")
    confidence_level: Literal["HIGH", "MEDIUM", "LOW"]
    # --- Domain-specific fields go here based on project type ---
    errors: list[str] = Field(default_factory=list, description="Processing errors if any")
```

**Required fields** (all agents must include):
- `summary` — human-readable result
- `confidence_score` — 0.0 to 1.0 numeric confidence
- `confidence_level` — HIGH, MEDIUM, or LOW categorical
- `errors` — list of processing errors (empty if none)

Add domain-specific fields based on the project type (e.g., `sources` for RAG, `classification` for data enrichment).

---

## Domain Skill (`skills/<name>-skill/skill.md`)

Skills are Markdown documents that define the agent's reasoning process:

```markdown
# <Agent Name> — Domain Skill

## Role
You are a specialized <domain> agent. Your sole responsibility is: <one sentence scope>.

## Input Contract
You receive: <describe input fields and their types>

## Instructions
<Step-by-step reasoning process. Be explicit:>
1. <First thing the agent should check/do>
2. <Second step — include decision criteria and thresholds>
3. <Edge cases and fallback behavior>

## Output Requirements
- Always return a JSON object matching the <OutputSchema> Pydantic model exactly
- Never return free-form prose as your final response
- If data is missing, set confidence_level to LOW and describe gaps in `errors`
- confidence_score must reflect actual certainty, not optimism
```

Skills are NOT system prompts — they are structured decision documents with explicit contracts.

---

## Agent Dockerfile

Foundry agent containers are lightweight — no multi-stage build needed:

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

MAF agents use `from_agent_framework().run()` which starts its own HTTP server on port 8000.

---

## Agent `requirements.txt`

```
agent-framework>=0.1.0
azure-ai-agentserver>=0.1.0
azure-identity>=1.17.0
pydantic>=2.7.0
opentelemetry-api>=1.24.0
azure-monitor-opentelemetry-exporter>=1.0.0b26
```

Add project-specific dependencies as needed (e.g., `azure-search-documents` for RAG agents).

---

## Two-Mode Dispatcher (`hosted_agents.py`)

The dispatcher automatically switches between Foundry (production) and direct HTTP (local dev):

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
    """Dispatch a request to a Foundry agent or local container."""
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

**Critical**: In `docker-compose.yml`, do NOT set `AZURE_AI_PROJECT_ENDPOINT` on the backend service — its absence enables local HTTP mode where the backend routes directly to agent containers.

---

## Registration Script (`scripts/register_agents.py`)

Called from the postprovision hook to register agents with Foundry:

```python
"""Register agents with Foundry — called in postprovision hook."""
import os
import yaml
from pathlib import Path
from azure.identity import DefaultAzureCredential

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

## RBAC Requirements

All container app managed identities (backend + agent containers) must have **both** roles on the Foundry account scope:

| Role | GUID | Purpose |
|---|---|---|
| Cognitive Services OpenAI User | `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd` | Call Azure OpenAI models |
| Azure AI User | `53ca9b11-8b9d-4b51-acae-26b3df39f6f0` | Access Foundry Agent Service |

These must be assigned in `role-assignments.bicep` for every managed identity that calls Foundry.

---

## Bicep Requirement

When U11=Yes, include `ai-foundry.bicep` in `infra/main.bicep`. This module (defined in `references/bicep-patterns.md`) provisions:
- Foundry account (`Microsoft.CognitiveServices/accounts` with `kind: 'AIServices'`)
- Foundry project (child resource)
- Model deployment(s) (chat model, and embedding model if applicable)

---

## Docker Compose — Agent Services

Add agent containers as services in `docker-compose.yml`:

```yaml
  <agent-name>:
    build: ./agents/<agent-name>
    ports:
      - "8001:8000"           # Sequential ports per agent: 8001, 8002, etc.
    environment:
      - AZURE_OPENAI_DEPLOYMENT_NAME=${AZURE_OPENAI_DEPLOYMENT_NAME:-gpt-4o}
      - APPLICATION_INSIGHTS_CONNECTION_STRING=${APPLICATION_INSIGHTS_CONNECTION_STRING:-}
      # NOTE: Do NOT set AZURE_AI_PROJECT_ENDPOINT — its absence enables local HTTP mode
```

The backend service must reference agent URLs for local routing:

```yaml
  backend:
    environment:
      - <AGENT_NAME>_URL=http://<agent-name>:8000
```

---

## Quality Checklist (Foundry Agents)

Run these checks for any project with U11=Yes:

- [ ] `agent.yaml` has meaningful `description:` (2-3 sentences, not a placeholder)
- [ ] `main.py` uses `default_options={"response_format": Schema}` for structured output
- [ ] `schemas.py` has `confidence_score`, `confidence_level`, `summary`, and `errors` fields
- [ ] `skill.md` defines role, input contract, step-by-step instructions, and output requirements
- [ ] `register_agents.py` passes `description=agent_def["description"]` to `create_version()`
- [ ] `hosted_agents.py` correctly switches between Foundry mode and direct HTTP mode
- [ ] `docker-compose.yml` does NOT set `AZURE_AI_PROJECT_ENDPOINT` on the backend (enables local mode)
- [ ] `ai-foundry.bicep` module included in `infra/main.bicep`
- [ ] RBAC includes both `Cognitive Services OpenAI User` and `Azure AI User` roles
- [ ] Postprovision hook calls `register_agents.py`
