# Azure Scaffold Wizard

A universal AI agent skill that scaffolds complete, production-ready Azure projects through adaptive questioning. Works with Claude Code, GitHub Copilot, Codex, Cursor, and any [Agent Skills](https://skills.sh)-compatible tool.

Install with a single command:

```bash
npx skills add microsoft/Foundry-AI-solution-templates-creation
```

---

## Table of Contents

- [What It Does](#what-it-does)
- [Supported Project Types](#supported-project-types)
- [Installation](#installation)
- [Usage](#usage)
- [What Gets Generated](#what-gets-generated)
- [How It Works](#how-it-works)
- [Architecture — Progressive Disclosure](#architecture--progressive-disclosure)
- [Workflow — The 6-Step Process](#workflow--the-6-step-process)
- [Unsupported Project Types — Fallback Behavior](#unsupported-project-types--fallback-behavior)
- [Known Limitations](#known-limitations)
- [Extending the Skill — Adding New Project Types](#extending-the-skill--adding-new-project-types)
- [Extending the Skill — Adding New Common Patterns](#extending-the-skill--adding-new-common-patterns)
- [Extending the Skill — Modifying Existing Patterns](#extending-the-skill--modifying-existing-patterns)
- [Project Structure](#project-structure)
- [Reference File Anatomy](#reference-file-anatomy)
- [Compatibility](#compatibility)
- [Contributing](#contributing)
- [License](#license)

---

## What It Does

When activated, this skill turns your AI coding agent into a **scaffolding wizard** that:

1. **Asks what you want to build** — presents 8 supported project types (RAG chatbot, multi-agent system, API, etc.)
2. **Asks dynamic qualifying questions** — universal questions (project name, language, Azure region, auth, etc.) plus type-specific questions (vector store for RAG, orchestration topology for multi-agent, triggers for Functions, etc.)
3. **Generates a complete project** — every source file, Azure infrastructure, deployment configuration, documentation, CI/CD, and Responsible AI docs

Every scaffolded project is fully deployable with `azd up`. No manual assembly required.

---

## Supported Project Types

The skill ships with **8 first-class project types**. Each has a dedicated reference file with curated questions, folder structures, source code patterns, Bicep modules, and quality checklists.

| # | Type | Reference File | Description |
|---|------|---------------|-------------|
| 1 | **RAG Chatbot** | `references/rag-chatbot.md` | Retrieval-Augmented Generation chatbot with vector search (Azure AI Search, Cosmos DB vCore, or pgvector), embedding pipeline, and streaming chat API |
| 2 | **Multi-Agent System** | `references/multi-agent.md` | Multiple AI agents coordinated by an orchestrator with parallel/sequential/batched topologies. Microsoft Foundry and MAF compatible. |
| 3 | **API Backend** | `references/api-backend.md` | REST or GraphQL API service with database, authentication middleware, pagination, and rate limiting |
| 4 | **Data Pipeline** | `references/data-pipeline.md` | ETL/ELT data processing pipeline — batch or streaming — with extractors, transformers, and loaders |
| 5 | **Azure Functions** | `references/function-app.md` | Serverless event-driven functions with HTTP, Timer, Queue, Blob, and Event Grid triggers. Supports Durable Functions. |
| 6 | **Full-Stack Web App** | `references/full-stack-app.md` | Frontend (Next.js, React SPA) + Backend (FastAPI, Express) with typed API client and shared models |
| 7 | **ML Training & Inference** | `references/ml-training.md` | Azure ML workspace with training scripts, pipeline definitions, model registry, and managed online endpoints |
| 8 | **Event-Driven Microservices** | `references/event-driven.md` | Message-based distributed system using Azure Service Bus and/or Event Grid with KEDA scaling and dead-letter handling |

---

## Installation

### Via skills.sh CLI (recommended)

```bash
npx skills add microsoft/Foundry-AI-solution-templates-creation
```

The CLI will prompt you to choose which agent(s) to install for. It copies the entire skill folder (`SKILL.md` + `references/`) into the agent's skills directory.

### Manual Installation

Clone this repository and copy the entire directory to your agent's skills folder:

```bash
git clone https://github.com/microsoft/Foundry-AI-solution-templates-creation.git
```

Then copy to the appropriate location:

| Agent | Project-Level Path | Global Path |
|-------|-------------------|-------------|
| **Claude Code** | `<project>/.claude/skills/azure-scaffold-wizard/` | `~/.claude/skills/azure-scaffold-wizard/` |
| **GitHub Copilot** | `<project>/.agents/skills/azure-scaffold-wizard/` | `~/.agents/skills/azure-scaffold-wizard/` |
| **Codex** | — | `~/.codex/skills/azure-scaffold-wizard/` |
| **Cursor** | `<project>/.agents/skills/azure-scaffold-wizard/` | `~/.cursor/skills/azure-scaffold-wizard/` |
| **Windsurf** | `<project>/.agents/skills/azure-scaffold-wizard/` | `~/.windsurf/skills/azure-scaffold-wizard/` |

**Project-level** installs apply only to that project. **Global** installs apply to all projects on your machine.

---

## Usage

After installation, simply ask your AI agent to scaffold a project using natural language:

```
"Scaffold a RAG chatbot that uses Azure AI Search and gpt-4o for legal document analysis"

"Create a multi-agent solution for insurance claims processing with 4 specialized agents"

"Build an API backend with Cosmos DB and Entra ID auth for a task management app"

"I need a data pipeline that ingests CSV files from blob storage, transforms them, and loads into Cosmos DB"

"Create a serverless Azure Functions app with queue triggers for order processing"

"Build a full-stack web app with Next.js frontend and FastAPI backend for a customer portal"

"Set up an ML training pipeline with Azure ML for a text classification model"

"Create an event-driven microservice architecture with Service Bus for an e-commerce platform"
```

The skill activates automatically when it detects a scaffolding request. It then:

1. Presents the project type selection
2. Asks qualifying questions (10 universal + 6-10 type-specific)
3. Generates every file in the project

### Deploying the Generated Project

Once the scaffold is generated:

```bash
cd <generated-project-name>
azd auth login
azd up
```

`azd up` provisions all Azure resources (from the Bicep templates) and deploys the application.

For local development without Azure:

```bash
docker compose up
```

---

## What Gets Generated

Every scaffolded project includes the following. None of these are stubs or placeholders — all files contain complete, functional content.

| Category | What's Included |
|----------|----------------|
| **Source Code** | Complete application code for all services — API endpoints, business logic, data models, configuration, tests |
| **Azure Bicep Infrastructure** | `infra/main.bicep` (subscription-scoped) + modular resource files: AI Foundry, Container Apps, ACR, AI Search, Cosmos DB, Storage, Key Vault, Service Bus, monitoring, RBAC |
| **Deployment Configuration** | `azure.yaml` with preprovision and postprovision hooks for both Windows (PowerShell) and Linux (bash) |
| **Docker** | Multi-stage `Dockerfile` per service + `docker-compose.yml` for full local development without Azure + `.dockerignore` |
| **CI/CD Workflows** | GitHub Actions: `ci.yml` (PR validation — lint, type-check, test, Bicep build) + `deploy.yml` (build, push to ACR, deploy on merge to main) |
| **README** | Project README with badges, architecture overview, quick deploy section, local dev instructions, component details, project structure tree |
| **Architecture Diagrams** | SVG solution architecture diagram + data/agent flow diagram (rendered inline in README) |
| **Responsible AI** | `TRANSPARENCY_FAQ.md` covering all 6 required questions, populated with project-specific content |
| **Supporting Docs** | `architecture.md`, `DeploymentGuide.md`, `production-migration.md`, `troubleshooting.md`, `requirements.md` |
| **Standard Files** | `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md`, `LICENSE`, `.env.example` |
| **Observability** | OpenTelemetry setup with Azure Monitor exporter, structured logging, trace correlation |

---

## How It Works

### Architecture — Progressive Disclosure

The skill uses a **progressive disclosure** pattern to stay efficient and context-aware:

```
SKILL.md (core orchestrator, ~400 lines)
    │
    ├── Always loaded first
    │   Contains: 6-step workflow, project type routing table,
    │   universal questions (U1-U10), quality checklist
    │
    ├── Loads ONE type-specific reference file based on user's choice
    │   e.g., references/rag-chatbot.md
    │   Contains: type-specific questions, folder structure,
    │   source file patterns, Bicep modules list, quality checklist
    │
    └── Loads common reference files as needed in Steps 4-5
        references/bicep-patterns.md    → Azure infrastructure
        references/azure-yaml-patterns.md → Deployment config
        references/docker-patterns.md    → Container setup
        references/readme-template.md    → Documentation
        references/cicd-patterns.md      → CI/CD workflows
        references/responsible-ai.md     → RAI documentation
        references/architecture-diagrams.md → SVG diagrams
        references/observability-patterns.md → Monitoring
        references/security-patterns.md  → Auth, RBAC, networking
```

**Why this matters**: The agent only loads reference files relevant to the user's choice. A user building a RAG chatbot never loads the multi-agent patterns, and vice versa. This keeps the agent's context focused and produces higher-quality output.

### Workflow — The 6-Step Process

| Step | Name | What Happens |
|------|------|-------------|
| **1** | **Identify & Gather Requirements** | Present 8 project types. Ask 10 universal questions (U1-U10). Load type-specific reference file and ask its questions (e.g., R1-R10 for RAG). Document all answers in `docs/requirements.md`. |
| **2** | **Generate Project Structure** | Create the complete folder layout — common root files + type-specific directories from the reference file. |
| **3** | **Generate Source Files** | Follow the type-specific reference file's generation instructions to produce all application code with complete, functional implementations. |
| **4** | **Generate Infrastructure & Deployment** | Load `bicep-patterns.md` → generate `main.bicep` + all modules. Load `azure-yaml-patterns.md` → generate `azure.yaml` + hooks. Load `docker-patterns.md` → generate Dockerfiles + compose. |
| **5** | **Generate Documentation** | Load `readme-template.md`, `architecture-diagrams.md`, `responsible-ai.md`, `cicd-patterns.md` → generate README, SVGs, TRANSPARENCY_FAQ, workflows, supporting docs. |
| **6** | **Quality Checklist** | Run universal checks (Bicep scoping, secrets, hooks, Docker, CI/CD, SVGs) + type-specific checks from the reference file. |

### Adaptive Questioning

Questions are asked in two phases:

1. **Universal questions (U1-U10)** — asked for ALL project types: business problem, end users, project name, programming language, Azure region, authentication, additional Azure services, frontend needed, compliance requirements, deployment platform.

2. **Type-specific questions** — asked only for the selected project type. For example:
   - **RAG Chatbot** (R1-R10): vector store, embedding model, chunking strategy, LLM, context window, conversation memory, streaming, reranking, top-k
   - **Multi-Agent** (T1-T10): agent names/roles, orchestration topology, MCP tools, model, regions, SKU
   - **API Backend** (A1-A8): endpoints, database, API style, pagination, rate limiting, versioning

Users only see questions relevant to their project type.

---

## Unsupported Project Types — Fallback Behavior

If a user requests a project type that is **not one of the 8 supported types** (e.g., "Build me a real-time IoT telemetry dashboard"), the skill still works but with reduced precision:

1. The agent maps the request to the **closest supported type** or **combines patterns** from multiple types (e.g., IoT dashboard = Event-Driven + Full-Stack Web App)
2. Universal questions (U1-U10) are still asked
3. The agent loads the closest reference file(s) and adapts the type-specific questions
4. The scaffold is generated using the adapted patterns

**What works well in fallback mode:**
- Universal infrastructure (Bicep, azure.yaml, Docker, CI/CD, README, Responsible AI docs)
- Common Azure service modules (monitoring, ACR, ACA, RBAC)
- Project structure conventions and deployment configuration

**What may be less precise in fallback mode:**
- Type-specific questions may not be perfectly tailored to the novel use case
- Source code patterns are adapted from the closest type rather than purpose-built
- Folder structure may not perfectly match the ideal layout for the novel type
- The quality checklist won't have type-specific validation items

**Recommendation**: If you frequently scaffold a project type that isn't supported, [add a new reference file](#extending-the-skill--adding-new-project-types) to make it a first-class citizen.

---

## Known Limitations

### Scaffold Limitations

| Limitation | Details |
|-----------|---------|
| **Not production-ready out of the box** | Generated scaffolds are solution accelerators — starting points. They require security hardening, load testing, and compliance review before production use. See the generated `docs/production-migration.md` for specifics. |
| **No authentication by default** | Scaffolds ship without auth enabled for simplicity. Production use requires Microsoft Entra ID integration (patterns documented in `references/security-patterns.md`). |
| **In-memory storage defaults** | Conversation memory (RAG), session state, and caching default to in-memory implementations that are lost on restart. Production requires persistent stores (Cosmos DB, Redis). |
| **English language only** | Generated code, prompts, and documentation are in English. Multilingual support is not scaffolded unless explicitly requested. |
| **Azure-only deployment** | All infrastructure targets Azure (Bicep, azd, ACA/Functions/AKS). AWS and GCP deployment are not supported. |
| **Python-first patterns** | While the skill asks about programming language (Python, TypeScript, C#), the reference file patterns are most detailed for Python. TypeScript and C# patterns are less comprehensive. |

### Skill Limitations

| Limitation | Details |
|-----------|---------|
| **8 supported project types** | Only the 8 listed types have dedicated reference files with curated patterns. Other types fall back to closest-match behavior (see [Fallback Behavior](#unsupported-project-types--fallback-behavior)). |
| **Agent context window** | Very large projects may strain the agent's context window. The progressive disclosure architecture mitigates this, but complex multi-service scaffolds with many components may require multiple agent sessions. |
| **No interactive preview** | The skill generates all files in one pass. There is no "preview and confirm" step between file generation. |
| **Reference file staleness** | Azure API versions, SDK versions, and best practices evolve. Reference files may contain outdated API versions or deprecated patterns over time. Regular maintenance is required. |
| **SVG diagram quality** | Architecture diagrams are generated as SVG by the LLM. Quality varies by agent — some agents produce better SVG than others. Manual touch-up may be needed. |
| **No incremental scaffolding** | The skill generates a new project from scratch each time. It does not modify existing projects or add features to previously scaffolded projects. |
| **No validation of Azure quotas** | The scaffold assumes sufficient Azure quota/capacity. It does not check if your subscription can provision the requested resources (e.g., GPU quota for ML training). |

### AI-Specific Limitations

| Limitation | Details |
|-----------|---------|
| **LLM output variability** | Different agents (Claude Code, Copilot, Codex) may produce slightly different scaffolds from the same questions, because each LLM interprets instructions differently. |
| **Hallucination risk** | While the reference files constrain the agent's output, the LLM may still generate code that references non-existent APIs, incorrect SDK methods, or wrong Bicep resource properties. Always review generated code. |
| **No execution or testing** | The skill instructs the agent to generate files but does not execute or test them. `az bicep build`, `docker compose config`, and test suites should be run manually after scaffolding. |

---

## Extending the Skill — Adding New Project Types

Adding a new project type is the most common extension. It requires **one new file** and **two table row edits**.

### Step 1: Create a Reference File

Create `references/<new-type>.md` with these required sections:

```markdown
# <Type Name> — Scaffold Pattern

Brief description of what this type scaffolds.

---

## Type-Specific Questions

| # | Question | Guidance |
|---|---|---|
| X1 | **Question text?** | How to interpret the answer. |
| X2 | ... | ... |

---

## Project Folder Structure

<project-slug>/
├── src/
│   └── ...
└── tests/
    └── ...

---

## Source File Generation Instructions

### <file-path>

(Code patterns with explanation of when to use which variant)

---

## Bicep Modules Required

- List of which Bicep modules from `references/bicep-patterns.md` this type needs
- Plus any new modules that need to be added

---

## Type-Specific Quality Checklist

- [ ] Check item 1
- [ ] Check item 2
```

Look at `references/rag-chatbot.md` or `references/api-backend.md` as templates.

### Step 2: Add to SKILL.md Routing Tables

In `SKILL.md`, add one row to **Step 1A** (Project Type Selection):

```markdown
| 9 | **<New Type>** | <Description> | <When to choose> |
```

And one row to **Step 1C** (Type-Specific Requirements):

```markdown
| <New Type> | `references/<new-type>.md` | <What questions it adds> |
```

### Step 3: Update `references/project-types.md`

Add the new type to the project types catalog with a full description.

### Examples of Types You Could Add

| Type | Reference File | Key Questions |
|------|---------------|---------------|
| IoT Telemetry | `references/iot-telemetry.md` | Device protocols (MQTT, AMQP), IoT Hub vs Event Hub, time-series DB, device provisioning |
| Chatbot with Plugins | `references/chatbot-plugins.md` | Plugin registry, tool calling patterns, safety filters, multi-turn memory |
| Document Intelligence | `references/doc-intelligence.md` | Document types (invoices, contracts, forms), AI Document Intelligence models, output format |
| Batch Inference | `references/batch-inference.md` | Input format, model type, parallelism, result storage, SLA |
| GraphRAG | `references/graphrag.md` | Knowledge graph store, entity extraction, community summarization, graph traversal |
| Real-Time Analytics | `references/real-time-analytics.md` | Stream source, windowing functions, dashboard framework, alerting |

---

## Extending the Skill — Adding New Common Patterns

Common patterns (in `references/`) are shared across all project types. To add a new common pattern:

1. Create `references/<new-pattern>.md` with the pattern content
2. Reference it from the appropriate step in `SKILL.md` (Step 4 for infrastructure, Step 5 for documentation)
3. Each type-specific reference file can optionally reference it

**Examples:**
- `references/api-management.md` — Azure API Management gateway patterns
- `references/redis-patterns.md` — Azure Cache for Redis integration
- `references/vnet-patterns.md` — Full VNet topology with private endpoints
- `references/terraform-patterns.md` — Terraform alternative to Bicep (if multi-IaC support is desired)

---

## Extending the Skill — Modifying Existing Patterns

To update existing patterns (e.g., newer Azure API versions, updated SDK methods):

1. Edit the relevant `references/*.md` file directly
2. Update API versions in `references/bicep-patterns.md` when Azure releases new stable versions
3. Update SDK patterns in type-specific files when libraries release breaking changes
4. Update CI/CD steps in `references/cicd-patterns.md` when GitHub Actions versions change

**Maintenance checklist** (recommended quarterly):
- [ ] Verify Bicep API versions are current
- [ ] Verify Python/TypeScript SDK versions in source patterns
- [ ] Verify GitHub Actions action versions (e.g., `actions/checkout@v4`)
- [ ] Verify Azure CLI / azd commands haven't changed
- [ ] Test at least one scaffold end-to-end per project type

---

## Project Structure

```
├── SKILL.md                              # Core skill: 6-step workflow + adaptive routing
│
├── references/
│   │
│   │  ── Type-specific patterns (one per project type) ──
│   │
│   ├── project-types.md                  # Catalog of all 8 project types with descriptions
│   ├── rag-chatbot.md                    # RAG: vector store, embedding, retrieval, chat
│   ├── multi-agent.md                    # Multi-agent: Foundry, orchestration topologies
│   ├── api-backend.md                    # API: CRUD, database, auth, rate limiting
│   ├── data-pipeline.md                  # Pipeline: ETL, extractors, transformers, loaders
│   ├── function-app.md                   # Functions: triggers, bindings, Durable Functions
│   ├── full-stack-app.md                 # Full-stack: Next.js + FastAPI, shared types
│   ├── ml-training.md                    # ML: training scripts, AML pipelines, endpoints
│   ├── event-driven.md                   # Event-driven: Service Bus, CloudEvents, KEDA
│   │
│   │  ── Common patterns (shared by all project types) ──
│   │
│   ├── bicep-patterns.md                 # Azure Bicep: main.bicep + all resource modules
│   ├── azure-yaml-patterns.md            # azure.yaml + preprovision/postprovision hooks
│   ├── readme-template.md                # README section template + conventions
│   ├── docker-patterns.md                # Dockerfiles + docker-compose patterns
│   ├── cicd-patterns.md                  # GitHub Actions CI/CD workflows
│   ├── responsible-ai.md                 # TRANSPARENCY_FAQ.md 6-question framework
│   ├── architecture-diagrams.md          # SVG generation prompts + Azure icon conventions
│   ├── observability-patterns.md         # OpenTelemetry + Azure Monitor setup
│   └── security-patterns.md              # Auth, RBAC, Key Vault, VNet patterns
│
├── README.md                             # This file
└── LICENSE                               # MIT
```

---

## Reference File Anatomy

Every reference file follows a consistent structure so agents can parse them reliably:

### Type-Specific Reference Files

| Section | Purpose |
|---------|---------|
| **Title + Description** | What this type scaffolds and when to use it |
| **Type-Specific Questions** | Table of questions (ID, question, guidance) asked only for this type |
| **Project Folder Structure** | Complete directory tree with annotations |
| **Source File Generation Instructions** | Code patterns per file, with variant selection based on question answers |
| **Bicep Modules Required** | Which modules from `bicep-patterns.md` to include, plus any new ones |
| **Type-Specific Quality Checklist** | Validation items specific to this project type |

### Common Reference Files

| Section | Purpose |
|---------|---------|
| **Design Principles** | Rules and conventions the agent must follow |
| **Templates** | Complete code/config templates with parameterized placeholders |
| **Variants** | Alternative implementations selected based on user answers |
| **Conventions** | Naming, formatting, and structural standards |

---

## Compatibility

### Tested Agents

| Agent | Status | Notes |
|-------|--------|-------|
| **Claude Code** | Supported | Full file-read and file-write capabilities. Best support for progressive disclosure (reads reference files on demand). |
| **GitHub Copilot** | Supported | Works via `.agents/skills/` directory. May have shorter context windows — progressive disclosure helps. |
| **OpenAI Codex** | Supported | Works via `~/.codex/skills/` directory. Strong code generation capabilities. |
| **Cursor** | Compatible | Uses `.agents/skills/` convention. |
| **Windsurf** | Compatible | Uses `.agents/skills/` convention. |
| **Other skills.sh agents** | Compatible | Any agent that supports the Agent Skills specification should work. |

### Requirements

- An AI coding agent with file-read and file-write capabilities
- Azure subscription (for deploying scaffolded projects)
- Azure CLI + Azure Developer CLI (`azd`) (for deployment)
- Docker (for local development)

---

## Contributing

We welcome contributions. The most impactful ways to contribute:

1. **Add a new project type** — follow the [extending guide](#extending-the-skill--adding-new-project-types)
2. **Improve existing patterns** — update API versions, fix bugs in code patterns, add missing edge cases
3. **Test with different agents** — verify the skill works correctly across Claude Code, Copilot, Codex, and others
4. **Report issues** — if a generated scaffold has bugs, incorrect Bicep, or missing files, open an issue

### Development Workflow

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/add-graphrag-type`
3. Make your changes
4. Test by installing the skill locally and running a scaffold
5. Submit a pull request

See [CONTRIBUTING.md](CONTRIBUTING.md) for full details.

---

## License

MIT — see [LICENSE](LICENSE).
