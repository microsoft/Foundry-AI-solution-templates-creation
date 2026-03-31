# Supported Project Types — Selection Guide

Use this catalog to help the user identify which project type best matches their needs. If the user's request spans multiple types, recommend the primary type and note which secondary patterns to incorporate.

---

## 1. RAG Chatbot

**What it is**: A conversational AI application that retrieves relevant information from a knowledge base (documents, databases, APIs) and uses an LLM to generate grounded responses.

**Key components**: Document ingestion pipeline, vector store, embedding model, Foundry RAG agent (default) or hand-rolled retrieval chain, chat API, optional frontend.

**Choose this when**:
- The user needs a chatbot that answers questions using their own data
- The user mentions "search over documents", "knowledge base", "Q&A system", or "grounded responses"
- The user mentions vector stores, embeddings, or RAG explicitly

**Azure services involved**: Azure AI Search or Cosmos DB vCore (vector store), Azure AI Foundry Agent Service (default for AI processing), Azure OpenAI (embeddings + LLM), Azure Container Apps, Azure Blob Storage (document storage).

**Reference file**: `references/rag-chatbot.md`

---

## 2. Multi-Agent System

**What it is**: A system where multiple specialized AI agents work together — either in parallel or sequentially — coordinated by an orchestrator. Each agent has a distinct responsibility, and a synthesis agent aggregates results. Designed for Microsoft Foundry Hosted Agents + Microsoft Agent Framework (MAF).

**Key components**: Agent containers (each with agent.yaml, main.py, schemas.py, skill.md), FastAPI orchestrator backend, two-mode dispatcher (local vs Foundry), optional frontend.

**Choose this when**:
- The user needs multiple AI agents collaborating on a complex task
- The user mentions "multi-agent", "agent pipeline", "orchestration", "Foundry", or "MAF"
- The domain involves parallel analysis with synthesis (e.g., insurance claims, medical review, compliance checking)

**Azure services involved**: Azure AI Foundry (agent hosting), Azure OpenAI, Azure Container Apps, Azure Container Registry.

**Reference file**: `references/multi-agent.md`

---

## 3. API Backend

**What it is**: A REST or GraphQL API service with database backing, authentication, and Azure deployment. Optionally includes AI inference endpoints powered by Azure AI Foundry Agent Service. Suitable for microservices, backend-for-frontend, or standalone API products.

**Key components**: FastAPI/Flask/Express routers, Pydantic/Zod schemas, database models, middleware (auth, CORS, rate limiting), health checks.

**Choose this when**:
- The user needs a backend API service
- The user mentions "REST API", "GraphQL", "CRUD", "microservice", or "backend"
- The primary output is an API consumed by other services or frontends

**Azure services involved**: Azure Container Apps, Azure Cosmos DB or PostgreSQL Flexible Server, Azure API Management (optional).

**Reference file**: `references/api-backend.md`

---

## 4. Data Pipeline

**What it is**: A batch or streaming data processing pipeline for ETL/ELT workloads. Optionally includes AI-powered data enrichment via Azure AI Foundry Agent Service. Transforms data from sources to sinks with scheduling, monitoring, and error handling.

**Key components**: Pipeline definitions, transformers, source/sink connectors, scheduling configuration, data quality checks.

**Choose this when**:
- The user needs to process, transform, or move data between systems
- The user mentions "ETL", "ELT", "data pipeline", "batch processing", or "streaming"
- The primary goal is data transformation rather than serving APIs or UIs

**Azure services involved**: Azure Data Factory, Azure Databricks or Synapse Spark, Azure Blob Storage / Data Lake Storage, Azure Event Hubs (for streaming).

**Reference file**: `references/data-pipeline.md`

---

## 5. Azure Functions

**What it is**: A serverless event-driven application using Azure Functions. Responds to triggers (HTTP, timer, queue, blob, Event Grid) without managing infrastructure. Optionally includes AI inference functions powered by Azure AI Foundry Agent Service.

**Key components**: Function triggers, input/output bindings, Durable Functions (for orchestration), shared utilities, local development settings.

**Choose this when**:
- The user needs lightweight, event-driven compute
- The user mentions "serverless", "Azure Functions", "triggers", or "event-driven" with simple processing
- Cost-efficiency and scale-to-zero are priorities

**Azure services involved**: Azure Functions (Consumption or Premium plan), Azure Storage, Azure Service Bus or Event Grid (triggers), Application Insights.

**Reference file**: `references/function-app.md`

---

## 6. Full-Stack Web App

**What it is**: A complete web application with frontend UI and backend API. Optionally includes AI features (chat, summarization, content generation) powered by Azure AI Foundry Agent Service. Includes authentication, database, and deployment as a cohesive unit.

**Key components**: Frontend (Next.js, React, or other), backend API, shared types/schemas, database models, authentication flow, static assets.

**Choose this when**:
- The user needs a complete web application with UI
- The user mentions "web app", "dashboard", "portal", "full-stack", or "frontend + backend"
- Both user-facing UI and backend logic are required

**Azure services involved**: Azure Container Apps (frontend + backend), Azure Cosmos DB or PostgreSQL, Azure Blob Storage (for uploads), Azure CDN (optional).

**Reference file**: `references/full-stack-app.md`

---

## 7. ML Training & Inference

**What it is**: A machine learning project covering the full model lifecycle: data preparation, training, evaluation, registration, and deployment as a managed inference endpoint.

**Key components**: Training scripts, model definitions, data loaders, evaluation metrics, inference endpoint, model registry integration.

**Choose this when**:
- The user needs to train and deploy a custom ML model
- The user mentions "model training", "inference endpoint", "Azure ML", "MLOps", or "fine-tuning"
- The primary goal is building/deploying a custom model, not using a pre-trained LLM via API

**Azure services involved**: Azure Machine Learning (workspace, compute, endpoints), Azure Blob Storage (datasets), Azure Container Registry (custom environments).

**Reference file**: `references/ml-training.md`

---

## 8. Event-Driven Microservices

**What it is**: A distributed system of decoupled services communicating through messages/events. Optionally includes AI-powered event processing via Azure AI Foundry Agent Service. Each service owns its data and responds to events asynchronously.

**Key components**: Message handlers, event publishers, event schemas, saga orchestrators, dead-letter processors, per-service databases.

**Choose this when**:
- The user needs an async, decoupled architecture at scale
- The user mentions "event-driven", "microservices", "message queue", "CQRS", or "saga pattern"
- Services need to evolve independently with eventual consistency

**Azure services involved**: Azure Service Bus or Event Grid, Azure Container Apps (per-service), Azure Cosmos DB (per-service), Azure Key Vault.

**Reference file**: `references/event-driven.md`

---

## Hybrid Patterns

Some projects combine multiple types. Common combinations:

| Primary Type | Secondary Pattern | Example |
|---|---|---|
| RAG Chatbot | API Backend | RAG system exposed as a reusable API for multiple frontends |
| Multi-Agent | RAG Chatbot | One or more agents use RAG for knowledge retrieval |
| Full-Stack Web App | API Backend | Standard web app with well-defined API layer |
| Event-Driven | Data Pipeline | Event-triggered data processing |
| API Backend | Azure Functions | API with some serverless background processing |
| Any AI-capable type | Foundry Agent | AI capabilities powered by hosted Foundry agent (default when AI features selected via U11) |

When combining types: use the primary type's folder structure as the base, then incorporate specific patterns from the secondary type's reference file.
