# Responsible AI Documentation — TRANSPARENCY_FAQ.md Template

Use this template when generating `TRANSPARENCY_FAQ.md` for any scaffolded project. The FAQ must address all 6 required questions. Populate every section using the requirements from Step 1 — no placeholder text.

---

## Template

```markdown
# Transparency FAQ — <Project Name>

## What is <Project Name>?

<Describe the solution accelerator, derived from U1. Include:>
- What it does (one paragraph)
- The technology stack (programming language from U4, Azure services from U7, deployment platform from U10)
- The domain and industry context (from U1/U2)

> **Important:** This is a solution accelerator — not a production-ready application. It is intended as a starting point for organizations building similar systems.

## What can <Project Name> do?

<Bullet list of capabilities, derived from the project type and type-specific requirements:>

- **Capability 1**: <Description> (e.g., "Ingests documents and indexes them in a vector store for retrieval")
- **Capability 2**: <Description> (e.g., "Generates grounded responses using retrieved context")
- **Capability N**: <Description>

<If multi-agent, list what each agent does:>
- **<Agent Name>**: <What it analyzes and produces>

## What are <Project Name>'s intended uses?

### Intended uses:
- Starting point for organizations building <type of system> on Azure
- Demonstrating <architectural pattern> patterns with Azure services
- Learning and experimentation with <technology>
- Internal proof-of-concept development

### NOT intended for:
- Autonomous decision-making without human oversight
- Production use without thorough testing, security review, and compliance validation
- Regulatory-compliance use without customer-specific validation
- Processing sensitive data without appropriate security hardening (see production-migration.md)

## How was <Project Name> evaluated?

<Describe evaluation approach, derived from the project type:>

- **Functional testing**: <What was tested — e.g., "Agent schema validation, decision logic, API endpoint correctness">
- **Structured output validation**: <e.g., "Pydantic model conformance for all agent outputs">
- **Integration testing**: <e.g., "End-to-end pipeline testing with sample data">
- **Infrastructure validation**: <e.g., "Bicep template validation with `az bicep build`">

> **Note:** This accelerator has NOT been evaluated for production-grade reliability, security, or compliance. Customers must conduct their own evaluation.

## What are the limitations of <Project Name>?

<List limitations, derived from the scaffold defaults and compliance requirements (U9):>

- **In-memory storage**: Data is lost on restart. Production requires a persistent database.
- **No authentication**: The default scaffold has no auth. Production requires Microsoft Entra ID.
- **English only**: Unless explicitly configured for multilingual support.
- **AI output requires review**: All AI-generated outputs must be reviewed by a human before any consequential action.
<If RAG:>
- **Data coverage gaps**: Retrieval quality depends entirely on the indexed documents. Missing or outdated documents produce incomplete answers.
- **Hallucination risk**: Despite RAG grounding, the LLM may generate content not present in the source documents.
<If multi-agent:>
- **External data sources**: MCP tools may have coverage gaps, rate limits, or stale data.
- **Agent coordination**: Complex multi-agent interactions may produce unexpected results at the boundaries.
<If applicable from U9:>
- **Compliance**: Compliance with <regulations from U9> is the **customer's responsibility**. This accelerator does not guarantee regulatory compliance.

## What operational factors and settings allow for effective and responsible use?

<List operational considerations:>

- **Model temperature**: Lower temperatures (0.1–0.3) produce more deterministic outputs. Higher temperatures increase creativity but reduce reliability.
- **Token limits**: Response quality degrades when context exceeds the model's effective window.
- **Latency**: Response times depend on model inference time, retrieval latency, and network conditions.
- **Scaling**: Azure Container Apps scaling rules affect cold-start latency and concurrent capacity.
- **Monitoring**: Azure Application Insights provides observability into performance, errors, and usage patterns.
- **Cost**: Token consumption, vector search queries, and compute time drive operational costs. Start with conservative limits and scale based on usage.
<If multi-agent:>
- **Agent timeouts**: Individual agent timeouts affect overall pipeline latency. Configure based on SLA requirements.
<If RAG:>
- **Chunk size and overlap**: Affect retrieval precision. Tune based on document types and query patterns.
- **Top-k retrieval count**: Higher values provide more context but increase cost and may introduce noise.
```

---

## TRANSPARENCY_FAQ.md Conventions

- Use the actual project name throughout (not generic placeholders)
- Every section must be populated with project-specific content
- Include the "Important" callout about solution accelerator status
- Link to `docs/production-migration.md` for production guidance
- Reference specific Azure services and technologies used
- Be honest about limitations — do not understate risks
- Include the customer responsibility statement for compliance topics

---

## Supporting Documents

In addition to TRANSPARENCY_FAQ.md, generate these standard files:

### `CODE_OF_CONDUCT.md`

```markdown
# Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).

For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
```

### `CONTRIBUTING.md`

```markdown
# Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a Contributor License Agreement (CLA).

For details, visit [https://cla.opensource.microsoft.com](https://cla.opensource.microsoft.com).

## Getting Started

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Make your changes
4. Run tests: `<test command based on U4>`
5. Submit a pull request

## Development Setup

See the [Local Development](#local-development) section in README.md.
```

### `SECURITY.md`

```markdown
# Security

## Reporting Security Issues

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, please report them via the repository's security advisories or by contacting the maintainers directly.

For more information, see [Microsoft's Security Response Center](https://www.microsoft.com/msrc).
```

### `SUPPORT.md`

```markdown
# Support

## How to Get Help

- **GitHub Issues**: For bug reports and feature requests
- **Discussions**: For questions and community interaction
- **Documentation**: See the [docs/](docs/) directory

## Microsoft Support Policy

Support for this project is limited to the resources listed above.
```
