# Azure Scaffold Wizard

A universal AI agent skill that scaffolds complete, production-ready Azure projects through adaptive questioning. Works with Claude Code, GitHub Copilot, Codex, Cursor, and any [Agent Skills](https://skills.sh)-compatible tool.

## What It Does

When activated, this skill:
1. Asks what you want to build (RAG chatbot, multi-agent system, API, etc.)
2. Asks dynamic qualifying questions based on your choice
3. Generates a **complete project** with all source files, Azure infrastructure, deployment configuration, documentation, and CI/CD

Every scaffolded project is deployable with `azd up`.

## Supported Project Types

| Type | Description |
|------|-------------|
| **RAG Chatbot** | Retrieval-Augmented Generation with vector search |
| **Multi-Agent System** | Coordinated AI agents with orchestration (Foundry compatible) |
| **API Backend** | REST/GraphQL API service with database |
| **Data Pipeline** | ETL/ELT batch or streaming processing |
| **Azure Functions** | Serverless event-driven functions |
| **Full-Stack Web App** | Frontend (Next.js) + Backend (FastAPI) |
| **ML Training & Inference** | Azure ML workspace + managed endpoints |
| **Event-Driven Microservices** | Message-based distributed system with Service Bus |

## Installation

```bash
npx skills add microsoft/Foundry-AI-solution-templates-creation
```

This installs the skill for your active AI agent (Claude Code, GitHub Copilot, Codex, etc.).

### Manual Installation

Clone this repo and copy the entire directory to your agent's skills folder:

| Agent | Project-Level Path | Global Path |
|-------|-------------------|-------------|
| **Claude Code** | `.claude/skills/azure-scaffold-wizard/` | `~/.claude/skills/azure-scaffold-wizard/` |
| **GitHub Copilot** | `.agents/skills/azure-scaffold-wizard/` | `~/.agents/skills/azure-scaffold-wizard/` |
| **Codex** | — | `~/.codex/skills/azure-scaffold-wizard/` |
| **Cursor** | `.agents/skills/azure-scaffold-wizard/` | `~/.cursor/skills/azure-scaffold-wizard/` |

## Usage

After installation, ask your AI agent to scaffold a project:

```
"Scaffold a RAG chatbot that uses Azure AI Search and gpt-4o"

"Create a multi-agent solution for insurance claims processing"

"Build an API backend with Cosmos DB for a task management app"

"I need a data pipeline that ingests CSV files from blob storage"

"Create a serverless Azure Functions app with queue triggers"
```

The skill activates automatically and guides you through the process.

## What Gets Generated

Every scaffolded project includes:

| Category | Files |
|----------|-------|
| **Source Code** | Complete, functional application code — not stubs or placeholders |
| **Azure Bicep** | `infra/main.bicep` + all resource modules (subscription-scoped, `azd`-compatible) |
| **Deployment** | `azure.yaml` with preprovision/postprovision hooks (Windows + Linux) |
| **Docker** | Multi-stage `Dockerfile` per service + `docker-compose.yml` for local dev |
| **CI/CD** | GitHub Actions workflows for PR validation and deployment |
| **Documentation** | README with architecture diagrams, deployment guide, troubleshooting |
| **Responsible AI** | `TRANSPARENCY_FAQ.md` covering all 6 required questions |
| **Observability** | OpenTelemetry + Azure Monitor integration |

## How It Works

The skill uses a **progressive disclosure** architecture:

1. **`SKILL.md`** — Core 6-step workflow that orchestrates the entire scaffolding process
2. **Type-specific reference files** (`references/rag-chatbot.md`, `references/multi-agent.md`, etc.) — Loaded on demand based on what the user wants to build. Contains type-specific questions, folder structures, and source file patterns.
3. **Common reference files** (`references/bicep-patterns.md`, `references/docker-patterns.md`, etc.) — Shared patterns used by all project types for infrastructure, deployment, and documentation.

The agent only loads the reference files relevant to the user's choice, keeping context efficient.

## Project Structure

```
├── SKILL.md                          # Core skill file
├── references/
│   ├── project-types.md              # Project type catalog
│   ├── rag-chatbot.md                # RAG chatbot patterns
│   ├── multi-agent.md                # Multi-agent patterns (Foundry)
│   ├── api-backend.md                # API backend patterns
│   ├── data-pipeline.md              # Data pipeline patterns
│   ├── function-app.md               # Azure Functions patterns
│   ├── full-stack-app.md             # Full-stack web app patterns
│   ├── ml-training.md                # ML training/inference patterns
│   ├── event-driven.md               # Event-driven microservices patterns
│   ├── bicep-patterns.md             # Azure Bicep infrastructure
│   ├── azure-yaml-patterns.md        # azure.yaml + hook scripts
│   ├── readme-template.md            # README generation template
│   ├── docker-patterns.md            # Docker configuration
│   ├── cicd-patterns.md              # CI/CD workflows
│   ├── responsible-ai.md             # Responsible AI documentation
│   ├── architecture-diagrams.md      # SVG diagram generation
│   ├── observability-patterns.md     # OpenTelemetry + Azure Monitor
│   └── security-patterns.md          # Auth, RBAC, Key Vault, VNet
├── README.md
└── LICENSE
```

## Extending

To add a new project type:

1. Create `references/<new-type>.md` following the pattern of existing type files (questions, folder structure, source patterns, Bicep modules, quality checklist)
2. Add one row to the **Project Type Selection** table in `SKILL.md` Step 1A
3. Add one row to the **Type-Specific Requirements** routing table in `SKILL.md` Step 1C

## License

MIT
