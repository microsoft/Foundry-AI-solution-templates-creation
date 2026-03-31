# README.md Generation Template

Use this template when generating the README.md for any scaffolded project. Populate every section using the requirements from Step 1. Do not leave placeholder text in the final output.

---

## Required Sections (in order)

### 1. H1 Title

Use the solution name derived from U1/U3.

### 2. Badges Row

```markdown
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Azure Deployable](https://img.shields.io/badge/Azure-Deployable-0078D4?logo=microsoftazure)](https://azure.microsoft.com)
<!-- Add technology-specific badges based on project type -->
```

Common badges by project type:
- RAG: `[![Azure AI Search](https://img.shields.io/badge/Azure_AI_Search-Enabled-purple)]`
- Multi-Agent: `[![Agent Framework](https://img.shields.io/badge/MAF-Agent_Framework-blue)]`
- Functions: `[![Azure Functions](https://img.shields.io/badge/Azure_Functions-Serverless-blue)]`

### 3. One-Paragraph Summary

Combine: U1 answer + technology stack + primary output + key differentiator. Keep to 3-4 sentences.

### 4. Responsible AI Callout

```markdown
> [!NOTE]
> This is a solution accelerator — not a production-ready application. AI-generated outputs require human review before any consequential action. For details, see [TRANSPARENCY_FAQ.md](TRANSPARENCY_FAQ.md) and [Microsoft Responsible AI principles](https://www.microsoft.com/en-us/ai/responsible-ai).
```

### 5. Navigation Anchors

```markdown
[SOLUTION OVERVIEW](#solution-overview) | [QUICK DEPLOY](#quick-deploy) | [BUSINESS SCENARIO](#business-scenario) | [LOCAL DEVELOPMENT](#local-development) | [SUPPORTING DOCS](#supporting-documentation)
```

### 6. Solution Overview

Include:
- SVG solution architecture diagram: `![Solution Architecture](docs/images/readme/solution-architecture.svg)`
- Pipeline/flow diagram (if applicable): `![Pipeline Flow](docs/images/readme/pipeline-flow.svg)`
- Key features as collapsible `<details>` blocks (one per major component + one for observability + one for local dev)

```markdown
<details>
<summary><strong>Feature Name</strong> — brief description</summary>

Detailed description of the feature, what it does, and how it works.
</details>
```

### 7. Runtime Modes Table

```markdown
## Runtime Modes

| Mode | How to Start | What Happens |
|------|-------------|--------------|
| **Azure (recommended)** | `azd up` | All services deployed to Azure; uses managed identities |
| **Local / Docker Compose** | `docker compose up` | All containers run locally — no Azure required |
```

For projects with Foundry agents (U11=Yes or Multi-Agent), add a callout explaining local routing:

```markdown
> [!TIP]
> In local mode, the backend routes directly to agent containers via Docker networking instead of through Azure AI Foundry. This happens automatically — no configuration changes needed.
```

### 8. Quick Deploy

Include:
- **Prerequisites checklist**: Azure subscription, `azd` CLI, Docker, language runtime (Python 3.12+, Node.js 20+, etc.)
- **Step-by-step** `azd up` walkthrough (numbered list)
- **Environment variables reference table**: name, required/optional, description, example value

```markdown
## Quick Deploy

### Prerequisites

- [ ] Azure subscription with contributor access
- [ ] [Azure Developer CLI (azd)](https://aka.ms/azd-install) >= 1.18.0
- [ ] Docker Desktop
- [ ] Python 3.12+ (or appropriate runtime)
- [ ] Node.js 20+ (if frontend included)

### Deploy to Azure

1. Clone the repository
2. `azd auth login`
3. `azd up`
4. Follow the prompts to select environment name and region

### Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `AZURE_ENV_NAME` | Yes | azd environment name | `my-project-dev` |
| `AZURE_LOCATION` | Yes | Azure region | `eastus2` |
<!-- Add all project-specific env vars -->
```

### 9. Business Scenario

Expand on U1–U2: who uses this solution, what problem it solves, what the workflow looks like. 2-3 paragraphs.

### 10. Local Development

- `.env.example` setup instructions
- `docker compose up` command
- How to test locally (curl example or frontend URL)
- Hot-reload configuration

### 11. Component Details

Table with all major components:

```markdown
| Component | Responsibility | Technology | Port |
|-----------|---------------|------------|------|
| Backend | API + orchestration | FastAPI | 8000 |
| Frontend | User interface | Next.js | 3000 |
<!-- Add project-specific components -->
```

For projects with Foundry agents (U11=Yes), add agent rows to the component table:

```markdown
| Component | Responsibility | Technology | Port |
|-----------|---------------|------------|------|
| Backend | API + orchestration | FastAPI | 8000 |
| <agent-name> | <agent description> | MAF (Foundry Agent) | 8001 |
| Frontend | User interface | Next.js | 3000 |
```

For multi-agent projects, also include an agent details table:

```markdown
| Agent | Responsibility | MCP Tools | Model | CPU/Memory |
|-------|---------------|-----------|-------|------------|
<!-- One row per agent -->
```

### 12. Project Structure

Condensed annotated tree — top 2 levels only:

```markdown
## Project Structure

\```
<project-slug>/
├── src/                    # Application source code
├── frontend/               # Frontend application (if applicable)
├── infra/                  # Azure Bicep infrastructure
├── scripts/                # Deployment hook scripts
├── docs/                   # Documentation
├── .github/workflows/      # CI/CD pipelines
├── azure.yaml              # azd project descriptor
├── docker-compose.yml      # Local development
└── README.md               # This file
\```
```

### 13. Extending the Solution

Brief guide: how to add a component (new agent, new endpoint, new service), how to swap a technology (change model, change database), how to add integrations.

### 14. Supporting Documentation

```markdown
| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | System architecture details |
| [Deployment Guide](docs/DeploymentGuide.md) | Step-by-step deployment |
| [Production Migration](docs/production-migration.md) | Moving from scaffold to production |
| [Troubleshooting](docs/troubleshooting.md) | Common issues and solutions |
| [API Reference](docs/api-reference.md) | API endpoint documentation |
```

### 15. Responsible AI

```markdown
## Responsible AI

This project follows [Microsoft Responsible AI principles](https://www.microsoft.com/en-us/ai/responsible-ai). See [TRANSPARENCY_FAQ.md](TRANSPARENCY_FAQ.md) for details on:
- What this solution can and cannot do
- Known limitations
- Evaluation methodology
- Operational considerations
```

---

## README Conventions

- All diagrams must be **SVG** (not PNG) — SVGs scale cleanly on GitHub and in dark mode
- Section anchors must be lowercase with hyphens (e.g., `#quick-deploy`)
- Interface screenshots (if included) must be 1600x900 PNG — scale to fit, no distortion
- Never use `<br>` for spacing — use blank lines between sections
- Use GitHub-flavored markdown alerts (`> [!NOTE]`, `> [!WARNING]`, `> [!TIP]`)
- Keep the README scannable — use tables, collapsible sections, and bullet points
