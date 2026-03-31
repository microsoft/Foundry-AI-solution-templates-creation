# Architecture Diagram Generation — SVG Patterns

Use these patterns and prompts when generating architecture diagrams for scaffolded projects.

---

## Two Diagrams Required

Every scaffolded project needs at least one SVG diagram. Most need two:

1. **`solution-architecture.svg`** — Azure resource topology showing all Azure services and their relationships
2. **`pipeline-flow.svg`** — Data/agent/request flow showing how data moves through the system (especially important for RAG, multi-agent, and data pipeline projects)

---

## SVG Generation Approach

Generate the SVG markup directly. The diagrams should be clean, minimal, and informative.

### Recommended Model for SVG Generation

**Gemini 3.1 Pro** produces significantly better SVG architecture diagrams than most other models. It generates cleaner layouts, more accurate spacing, proper arrow routing, and better visual hierarchy.

If the agent has access to Gemini 3.1 Pro (via API, MCP tool, or model switching):
- **Prefer Gemini 3.1 Pro** for all SVG diagram generation
- Pass it the prompt templates below along with the project requirements
- Review the output for correctness (services, arrows, labels) before saving

If Gemini 3.1 Pro is not available:
- Generate the SVG using the current agent's capabilities
- Follow the skeleton template and layout guidelines below closely — they compensate for weaker SVG generation by providing precise structure
- Note in the generated README that diagrams may benefit from manual refinement
- The user can regenerate diagrams later using a Gemini-capable tool

### Solution Architecture Diagram Prompt Template

Adapt this template based on the project type and services:

```
Generate a clean SVG architecture diagram showing an Azure-native <project-type> solution with these components:

- User/browser on the left
- <List compute services: e.g., "Azure Container Apps hosting: Backend API (FastAPI), Frontend (Next.js)">
- <List AI services if applicable: e.g., "Azure OpenAI (gpt-4o) for embeddings and generation">
- <List data services: e.g., "Azure AI Search (vector store), Azure Blob Storage (documents)">
- <List infrastructure services: e.g., "Azure Container Registry, Azure Application Insights, Azure Monitor">
- <List any additional services from U7>

Show data flow arrows:
- User → Frontend → Backend → <service chain based on project type>
- Cross-cutting observability arrows from all compute to App Insights

Style requirements:
- Clean, minimal design
- Dark background (#1E1E1E) with white labels
- Azure blue (#0078D4) for Azure services
- Purple (#7B2D8E) for AI/ML services
- Green (#107C10) for data services
- Rounded rectangles for service boxes
- Directional arrows for data flow
- Service icons or labels inside each box
```

### Pipeline Flow Diagram Prompt Template

```
Generate a clean SVG flow diagram showing the <project-type> pipeline:

<Customize based on project type:>

For RAG:
Input Document → Loader → Chunker → Embedder → Vector Store (indexing path)
User Query → Embed Query → Vector Search → Reranker → Context → LLM → Response (query path)

For Multi-Agent:
Input → [Agent 1 (parallel)] [Agent 2 (parallel)] → [Agent 3 (sequential)] → Synthesis Agent → Output
Show parallel execution with fork/join visual.

For Data Pipeline:
Source → Extract → Transform → Validate → Load → Sink
Show parallel branches if applicable.

For Event-Driven:
Publisher → Event Grid/Service Bus → Handler 1, Handler 2 → Store
Show async message flow.

Style: clean, minimal, color-coded boxes for each stage, directional arrows.
```

---

## SVG Technical Requirements

Every generated SVG must comply with these rules:

1. **Responsive scaling**: Use `viewBox` with `preserveAspectRatio="xMidYMid meet"`
   ```xml
   <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 600"
        preserveAspectRatio="xMidYMid meet" width="100%">
   ```

2. **No external dependencies**: No external font imports, no JavaScript, no external images
   ```xml
   <!-- CORRECT -->
   <text font-family="system-ui, -apple-system, 'Segoe UI', sans-serif">

   <!-- WRONG — do not use -->
   <style>@import url('https://fonts.googleapis.com/...')</style>
   ```

3. **Embedded styles only**: All CSS must be inline or in a `<style>` block within the SVG

4. **GitHub-compatible**: Must render correctly on GitHub's SVG renderer
   - No `<foreignObject>` elements
   - No scripts or event handlers
   - Simple shapes: `<rect>`, `<circle>`, `<path>`, `<text>`, `<line>`, `<polygon>`

5. **File size**: Target under 50KB per SVG

6. **Color palette**:
   | Element | Color | Hex |
   |---------|-------|-----|
   | Background | Dark gray | `#1E1E1E` |
   | Azure services | Azure blue | `#0078D4` |
   | AI/ML services | Purple | `#7B2D8E` |
   | Data services | Green | `#107C10` |
   | Text labels | White | `#FFFFFF` |
   | Arrows | Light gray | `#CCCCCC` |
   | Borders | Medium gray | `#404040` |

7. **Accessibility**: Include `<title>` and `<desc>` elements
   ```xml
   <svg ...>
     <title>Solution Architecture — <Project Name></title>
     <desc>Azure architecture diagram showing the components and data flow of the <project-type> solution.</desc>
   ```

---

## SVG Skeleton Template

Use this as a starting point and customize:

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 600"
     preserveAspectRatio="xMidYMid meet" width="100%">

  <title>Solution Architecture — <Project Name></title>
  <desc>Architecture diagram for the <project-type> solution on Azure.</desc>

  <style>
    .bg { fill: #1E1E1E; }
    .box { rx: 8; ry: 8; stroke-width: 1.5; }
    .azure { fill: #0078D4; stroke: #005A9E; }
    .ai { fill: #7B2D8E; stroke: #5B1D6E; }
    .data { fill: #107C10; stroke: #0B5C0B; }
    .label { fill: #FFFFFF; font-family: system-ui, -apple-system, sans-serif; font-size: 13px; text-anchor: middle; dominant-baseline: central; }
    .title { fill: #FFFFFF; font-family: system-ui, -apple-system, sans-serif; font-size: 11px; text-anchor: middle; font-weight: bold; }
    .arrow { stroke: #CCCCCC; stroke-width: 1.5; fill: none; marker-end: url(#arrowhead); }
    .section-label { fill: #888888; font-family: system-ui, sans-serif; font-size: 10px; text-anchor: start; }
  </style>

  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#CCCCCC" />
    </marker>
  </defs>

  <!-- Background -->
  <rect class="bg" width="1200" height="600" />

  <!-- Add service boxes, labels, and arrows here -->
  <!-- Group related services with <g> elements -->
  <!-- Use consistent spacing: 160px between columns, 80px between rows -->

</svg>
```

---

## Diagram Layout Guidelines

- **Left to right**: Data flows from left (user/input) to right (output/storage)
- **Top to bottom for layers**: Infrastructure at bottom, compute in middle, user at top
- **Consistent spacing**: 160px between columns, 80px between rows
- **Group related services**: Use dotted borders to group services in the same resource group or logical tier
- **Label all arrows**: Brief description of what data flows (e.g., "HTTP/JSON", "Embeddings", "Events")
- **Maximum 12 boxes**: Keep the diagram scannable — consolidate if needed
