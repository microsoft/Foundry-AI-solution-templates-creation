# RAG Chatbot — Scaffold Pattern

This reference file defines the scaffold pattern for **Retrieval-Augmented Generation (RAG) chatbot** applications deployed on Azure. It produces a system that ingests documents into a vector store, retrieves relevant context at query time, and generates grounded responses using an LLM.

---

## Type-Specific Questions

Ask these in addition to the universal questions (U1–U10):

| # | Question | Guidance |
|---|---|---|
| R1 | **What data sources will the RAG system index?** | Documents (PDF, Word, HTML), databases, APIs, web pages. Drives ingestion pipeline design. |
| R2 | **Which vector store?** | `Azure AI Search` (default, recommended), `Cosmos DB vCore` (MongoDB API), `PostgreSQL pgvector`. |
| R3 | **Which embedding model?** | `text-embedding-3-large` (default, 3072 dims), `text-embedding-3-small` (1536 dims), `text-embedding-ada-002` (1536 dims). |
| R4 | **Which LLM for generation?** | `gpt-4o` (default), `gpt-4.1`, `gpt-5.4`. Model choice constrains Azure region. |
| R5 | **Chunking strategy?** | `semantic` (default — splits by meaning boundaries), `fixed-size` (token-count windows), `recursive` (hierarchical splitting), `document-aware` (respects document structure like headings). |
| R6 | **Maximum context window for retrieval?** | Default: 8000 tokens. Drives chunk size and top-k retrieval count. |
| R7 | **Conversation memory?** | `in-memory` (default, lost on restart), `Cosmos DB` (persistent), `Redis` (fast, ephemeral). |
| R8 | **Streaming responses?** | `yes` (default — SSE for real-time chat UX), `no` (batch/synchronous). |
| R9 | **Reranking?** | `none` (default), `Azure AI Search semantic ranker`, `cross-encoder reranker` (local model). |
| R10 | **Number of retrieval results (top-k)?** | Default: 5. Higher values give more context but increase latency and cost. |
| R11 | **Use Azure AI Foundry Agent Service for chat/retrieval?** | **Yes** (default): Foundry RAG agent handles retrieval grounding and response generation via MAF. Ingestion pipeline (load → chunk → embed → index) still generated. **No**: hand-rolled retrieval chain with direct Azure OpenAI SDK calls. Maps to U11. |

---

## Project Folder Structure

```
<project-slug>/
├── src/
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── chunker.py              # Document chunking (strategy from R5)
│   │   ├── embedder.py             # Embedding generation (model from R3)
│   │   ├── loader.py               # Document loading from various formats (from R1)
│   │   └── pipeline.py             # End-to-end ingestion orchestration
│   │
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── vector_store.py         # Vector store client (from R2)
│   │   ├── reranker.py             # Optional reranking (from R9)
│   │   └── retriever.py            # Retrieval chain: query → embed → search → rerank → context
│   │
│   ├── chat/
│   │   ├── __init__.py
│   │   ├── chain.py                # RAG chain: retrieval + LLM generation
│   │   ├── prompts.py              # System and user prompt templates
│   │   └── memory.py               # Conversation memory (from R7)
│   │
│   └── api/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory + lifespan
│       ├── config.py               # pydantic-settings Settings class
│       ├── observability.py        # OTel + Azure Monitor setup
│       └── routers/
│           ├── __init__.py
│           ├── chat.py             # POST /chat (streaming SSE or sync JSON)
│           ├── ingest.py           # POST /ingest (upload + index documents)
│           └── health.py           # GET /health
│
├── frontend/                       # Include if U8=yes
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── next.config.ts
│   ├── package.json
│   ├── tsconfig.json
│   ├── app/
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── chat-interface.tsx      # Chat UI with message history
│   │   ├── message-bubble.tsx      # Individual message with source citations
│   │   ├── source-panel.tsx        # Expandable source document references
│   │   ├── upload-panel.tsx        # Document upload for ingestion
│   │   ├── header.tsx
│   │   └── ui/                     # shadcn/ui primitives
│   └── lib/
│       ├── api.ts                  # Typed fetch + SSE client
│       ├── types.ts                # TypeScript interfaces
│       └── utils.ts
│
├── data/                           # Sample documents for testing
│   └── sample/
│       └── README.md               # Instructions for adding sample docs
│
└── tests/
    ├── __init__.py
    ├── test_chunker.py
    ├── test_embedder.py
    ├── test_retriever.py
    ├── test_chain.py
    └── conftest.py                 # Shared fixtures (mock vector store, mock LLM)
```

### Folder Structure Changes — Foundry Mode (R11 = Yes)

When R11 = Yes, the following changes apply to the folder structure above:

- **REMOVE**: `src/chat/` directory entirely (chain.py, prompts.py, memory.py) — replaced by Foundry agent
- **ADD**: `agents/rag-agent/` directory (Foundry hosted agent)
- **ADD**: `src/api/agents/` directory (two-mode dispatcher)
- **ADD**: `scripts/register_agents.py` (Foundry agent registration)

```
<project-slug>/
├── agents/
│   └── rag-agent/
│       ├── agent.yaml                  # Foundry Hosted Agent descriptor
│       ├── Dockerfile
│       ├── main.py                     # MAF entry point (from_agent_framework)
│       ├── requirements.txt
│       ├── schemas.py                  # RAGResponse Pydantic model
│       └── skills/
│           └── rag-skill/
│               └── skill.md            # RAG domain rules and citation instructions
│
├── src/
│   ├── ingestion/                      # UNCHANGED — still needed to populate vector store
│   │   ├── __init__.py
│   │   ├── chunker.py
│   │   ├── embedder.py
│   │   ├── loader.py
│   │   └── pipeline.py
│   │
│   ├── retrieval/                      # UNCHANGED — vector store client still needed
│   │   ├── __init__.py
│   │   ├── vector_store.py
│   │   ├── reranker.py
│   │   └── retriever.py
│   │
│   └── api/
│       ├── __init__.py
│       ├── main.py                     # FastAPI app factory (no chat chain import)
│       ├── config.py
│       ├── observability.py
│       ├── agents/
│       │   ├── __init__.py
│       │   └── hosted_agents.py        # Two-mode dispatcher (local vs Foundry)
│       └── routers/
│           ├── __init__.py
│           ├── chat.py                 # MODIFIED: calls dispatcher instead of chain
│           ├── ingest.py
│           └── health.py
│
├── scripts/
│   └── register_agents.py             # Foundry agent registration
│
└── (frontend, data, tests — same as base structure)
```

---

## Source File Generation Instructions

The **Ingestion** and **Retrieval** sections below are generated for BOTH modes (Foundry and hand-rolled). The **Chat** section differs based on R11.

### Ingestion Pipeline (Both Modes)

#### `src/ingestion/loader.py`

```python
"""Document loader — supports multiple formats based on R1 answers."""
from pathlib import Path
from dataclasses import dataclass


@dataclass
class Document:
    content: str
    metadata: dict
    source: str


def load_pdf(file_path: Path) -> list[Document]:
    """Load a PDF file and extract text per page."""
    import fitz  # PyMuPDF

    docs = []
    with fitz.open(file_path) as pdf:
        for page_num, page in enumerate(pdf):
            text = page.get_text()
            if text.strip():
                docs.append(Document(
                    content=text,
                    metadata={"page": page_num + 1, "format": "pdf"},
                    source=str(file_path),
                ))
    return docs


def load_text(file_path: Path) -> list[Document]:
    """Load a plain text or markdown file."""
    content = file_path.read_text(encoding="utf-8")
    return [Document(
        content=content,
        metadata={"format": file_path.suffix.lstrip(".")},
        source=str(file_path),
    )]


def load_documents(path: Path) -> list[Document]:
    """Load all documents from a directory, dispatching by file type."""
    loaders = {
        ".pdf": load_pdf,
        ".txt": load_text,
        ".md": load_text,
        # Add more loaders based on R1 answers
    }
    docs = []
    for file_path in sorted(path.rglob("*")):
        if file_path.suffix.lower() in loaders:
            docs.extend(loaders[file_path.suffix.lower()](file_path))
    return docs
```

#### `src/ingestion/chunker.py`

Implement based on R5 answer:

```python
"""Document chunker — strategy selected based on R5."""
from dataclasses import dataclass


@dataclass
class Chunk:
    content: str
    metadata: dict
    chunk_index: int


def chunk_fixed_size(text: str, chunk_size: int = 512, overlap: int = 64) -> list[str]:
    """Fixed-size token-count windows with overlap."""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        if chunk.strip():
            chunks.append(chunk)
    return chunks


def chunk_recursive(text: str, max_size: int = 512) -> list[str]:
    """Recursive splitting by paragraph, then sentence, then word boundary."""
    separators = ["\n\n", "\n", ". ", " "]
    return _recursive_split(text, separators, max_size)


def _recursive_split(text: str, separators: list[str], max_size: int) -> list[str]:
    if len(text.split()) <= max_size:
        return [text] if text.strip() else []

    sep = separators[0] if separators else " "
    parts = text.split(sep)
    chunks = []
    current = ""

    for part in parts:
        candidate = f"{current}{sep}{part}" if current else part
        if len(candidate.split()) > max_size and current:
            chunks.append(current.strip())
            current = part
        else:
            current = candidate

    if current.strip():
        if len(current.split()) > max_size and len(separators) > 1:
            chunks.extend(_recursive_split(current, separators[1:], max_size))
        else:
            chunks.append(current.strip())

    return chunks


def chunk_documents(documents: list, strategy: str = "recursive", **kwargs) -> list[Chunk]:
    """Chunk a list of Documents using the selected strategy."""
    strategies = {
        "fixed-size": chunk_fixed_size,
        "recursive": chunk_recursive,
        # Add "semantic" and "document-aware" based on R5
    }
    chunker = strategies.get(strategy, chunk_recursive)

    all_chunks = []
    for doc in documents:
        text_chunks = chunker(doc.content, **kwargs)
        for i, text in enumerate(text_chunks):
            all_chunks.append(Chunk(
                content=text,
                metadata={**doc.metadata, "source": doc.source},
                chunk_index=i,
            ))
    return all_chunks
```

#### `src/ingestion/embedder.py`

```python
"""Embedding generator using Azure OpenAI."""
import os
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider


def get_embeddings_client() -> AzureOpenAI:
    """Create Azure OpenAI client for embeddings using managed identity."""
    token_provider = get_bearer_token_provider(
        DefaultAzureCredential(),
        "https://cognitiveservices.azure.com/.default",
    )
    return AzureOpenAI(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        azure_ad_token_provider=token_provider,
        api_version="2024-10-21",
    )


def embed_texts(texts: list[str], model: str = "<embedding-model-from-R3>") -> list[list[float]]:
    """Generate embeddings for a batch of texts."""
    client = get_embeddings_client()
    # Process in batches of 16 to stay within API limits
    all_embeddings = []
    batch_size = 16
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        response = client.embeddings.create(input=batch, model=model)
        all_embeddings.extend([item.embedding for item in response.data])
    return all_embeddings
```

#### `src/ingestion/pipeline.py`

```python
"""End-to-end ingestion pipeline: load → chunk → embed → index."""
from pathlib import Path
from .loader import load_documents
from .chunker import chunk_documents
from .embedder import embed_texts
from ..retrieval.vector_store import get_vector_store


async def ingest(source_path: Path, strategy: str = "recursive") -> dict:
    """Run the full ingestion pipeline."""
    # 1. Load documents
    documents = load_documents(source_path)

    # 2. Chunk documents
    chunks = chunk_documents(documents, strategy=strategy)

    # 3. Generate embeddings
    texts = [c.content for c in chunks]
    embeddings = embed_texts(texts)

    # 4. Index in vector store
    store = get_vector_store()
    indexed_count = await store.upsert(chunks, embeddings)

    return {
        "documents_loaded": len(documents),
        "chunks_created": len(chunks),
        "vectors_indexed": indexed_count,
    }
```

### Retrieval (Both Modes)

#### `src/retrieval/vector_store.py`

Generate based on R2 answer. Example for Azure AI Search:

```python
"""Vector store client — Azure AI Search implementation."""
import os
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex,
    SearchField,
    SearchFieldDataType,
    VectorSearch,
    HnswAlgorithmConfiguration,
    VectorSearchProfile,
    SearchableField,
    SimpleField,
)
from azure.identity import DefaultAzureCredential


INDEX_NAME = os.environ.get("AZURE_SEARCH_INDEX_NAME", "rag-index")


def get_index_client() -> SearchIndexClient:
    return SearchIndexClient(
        endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
        credential=DefaultAzureCredential(),
    )


def get_search_client() -> SearchClient:
    return SearchClient(
        endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
        index_name=INDEX_NAME,
        credential=DefaultAzureCredential(),
    )


def ensure_index(dimensions: int = 3072):
    """Create the search index if it doesn't exist."""
    client = get_index_client()
    fields = [
        SimpleField(name="id", type=SearchFieldDataType.String, key=True),
        SearchableField(name="content", type=SearchFieldDataType.String),
        SimpleField(name="source", type=SearchFieldDataType.String, filterable=True),
        SearchField(
            name="embedding",
            type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
            searchable=True,
            vector_search_dimensions=dimensions,
            vector_search_profile_name="default-profile",
        ),
    ]
    vector_search = VectorSearch(
        algorithms=[HnswAlgorithmConfiguration(name="default-algo")],
        profiles=[VectorSearchProfile(name="default-profile", algorithm_configuration_name="default-algo")],
    )
    index = SearchIndex(name=INDEX_NAME, fields=fields, vector_search=vector_search)
    client.create_or_update_index(index)


class AzureAISearchStore:
    """Vector store backed by Azure AI Search."""

    def __init__(self):
        self.client = get_search_client()

    async def upsert(self, chunks, embeddings) -> int:
        """Upsert chunks with their embeddings into the index."""
        documents = []
        for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
            documents.append({
                "id": f"{chunk.metadata.get('source', 'unknown')}_{chunk.chunk_index}",
                "content": chunk.content,
                "source": chunk.metadata.get("source", ""),
                "embedding": embedding,
            })
        result = self.client.upload_documents(documents)
        return len([r for r in result if r.succeeded])

    async def search(self, query_embedding: list[float], top_k: int = 5) -> list[dict]:
        """Search for similar documents using vector similarity."""
        from azure.search.documents.models import VectorizedQuery

        vector_query = VectorizedQuery(
            vector=query_embedding,
            k_nearest_neighbors=top_k,
            fields="embedding",
        )
        results = self.client.search(
            search_text=None,
            vector_queries=[vector_query],
            top=top_k,
        )
        return [
            {"content": r["content"], "source": r["source"], "score": r["@search.score"]}
            for r in results
        ]


def get_vector_store():
    """Factory function — swap implementation based on R2."""
    return AzureAISearchStore()
```

#### `src/retrieval/retriever.py`

```python
"""Retrieval chain: query → embed → search → rerank → context."""
from ..ingestion.embedder import embed_texts
from .vector_store import get_vector_store
from .reranker import rerank


async def retrieve(query: str, top_k: int = 5, use_reranker: bool = False) -> list[dict]:
    """Retrieve relevant documents for a query."""
    # 1. Embed the query
    query_embedding = embed_texts([query])[0]

    # 2. Vector search
    store = get_vector_store()
    results = await store.search(query_embedding, top_k=top_k * 2 if use_reranker else top_k)

    # 3. Optional reranking
    if use_reranker and results:
        results = rerank(query, results, top_k=top_k)

    return results[:top_k]


def format_context(results: list[dict]) -> str:
    """Format retrieved results into a context string for the LLM."""
    context_parts = []
    for i, result in enumerate(results, 1):
        source = result.get("source", "Unknown")
        content = result["content"]
        context_parts.append(f"[Source {i}: {source}]\n{content}")
    return "\n\n---\n\n".join(context_parts)
```

### Chat — Foundry Mode (R11 = Yes)

When R11 = Yes, the `src/chat/` directory is NOT generated. Instead, generate the Foundry agent files below. Use patterns from `references/foundry-agent-patterns.md` as the base.

#### `agents/rag-agent/agent.yaml`

```yaml
name: rag-agent
description: >
  RAG chatbot agent that retrieves relevant context from an Azure AI Search
  index and generates grounded, cited responses. Uses structured output
  to return answer text, source citations, and confidence scoring.
runtime: agent-framework
version: "1.0.0"
resources:
  cpu: "1"
  memory: "2Gi"
env:
  - name: AZURE_AI_PROJECT_ENDPOINT
    secretRef: azure-ai-project-endpoint
  - name: AZURE_OPENAI_DEPLOYMENT_NAME
    value: <model-from-R4>
  - name: AZURE_SEARCH_ENDPOINT
    secretRef: azure-search-endpoint
  - name: AZURE_SEARCH_INDEX_NAME
    value: <index-name-from-project>
  - name: APPLICATION_INSIGHTS_CONNECTION_STRING
    secretRef: app-insights-connection-string
```

#### `agents/rag-agent/schemas.py`

```python
from pydantic import BaseModel, Field
from typing import Literal


class RAGResponse(BaseModel):
    """Structured output for the RAG agent."""

    answer: str = Field(description="The grounded response to the user's question")
    sources: list[str] = Field(description="List of source document references cited in the answer")
    confidence_score: float = Field(ge=0.0, le=1.0, description="Confidence in answer quality 0-1")
    confidence_level: Literal["HIGH", "MEDIUM", "LOW"]
    summary: str = Field(description="One-sentence summary of the answer")
    follow_up_questions: list[str] = Field(
        default_factory=list, description="Suggested follow-up questions based on context"
    )
    errors: list[str] = Field(default_factory=list, description="Processing errors if any")
```

#### `agents/rag-agent/skills/rag-skill/skill.md`

```markdown
# RAG Agent — Domain Skill

## Role
You are a retrieval-augmented generation agent. Your sole responsibility is: answer user questions using ONLY the context retrieved from the knowledge base.

## Input Contract
You receive:
- `question` (string): The user's question
- `context` (string): Retrieved context passages with source metadata
- `session_id` (string): Conversation session identifier

## Instructions
1. Read all retrieved context passages carefully
2. Identify passages relevant to the user's question
3. Compose a grounded answer using ONLY information from the context
4. Cite sources using [Source N] notation for every factual claim
5. If the context is insufficient to fully answer, say so clearly and set confidence_level to LOW
6. Never fabricate information not present in the context
7. If multiple sources agree, set confidence_level to HIGH
8. If sources conflict, note the discrepancy and set confidence_level to MEDIUM
9. Suggest 1-3 relevant follow-up questions based on the available context

## Output Requirements
- Always return a JSON object matching the RAGResponse schema exactly
- Never return free-form prose as your final response
- confidence_score must reflect actual grounding quality, not optimism
- sources list must contain all referenced document identifiers
- If no relevant context found, set confidence_level=LOW and explain in answer
```

#### `agents/rag-agent/main.py`

Follow the MAF entry point pattern from `references/foundry-agent-patterns.md`:

```python
"""RAG Agent — Foundry Hosted Agent entry point."""
import os
from agent_framework import SkillsProvider
from agent_framework.azure import AzureOpenAIResponsesClient
from azure.ai.agentserver.agentframework import from_agent_framework
from azure.identity import DefaultAzureCredential
from schemas import RAGResponse


def main() -> None:
    _ai_conn = os.environ.get("APPLICATION_INSIGHTS_CONNECTION_STRING")
    if _ai_conn:
        os.environ.setdefault("APPLICATIONINSIGHTS_CONNECTION_STRING", _ai_conn)

    skills = SkillsProvider.from_directory("skills")

    client = AzureOpenAIResponsesClient(
        endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        model=os.environ.get("AZURE_OPENAI_DEPLOYMENT_NAME", "gpt-4o"),
    )

    from_agent_framework(
        client=client,
        skills=skills,
        default_options={"response_format": RAGResponse},
    ).run()


if __name__ == "__main__":
    main()
```

#### `agents/rag-agent/requirements.txt`

```
agent-framework>=0.1.0
azure-ai-agentserver>=0.1.0
azure-identity>=1.17.0
pydantic>=2.7.0
opentelemetry-api>=1.24.0
azure-monitor-opentelemetry-exporter>=1.0.0b26
```

#### `agents/rag-agent/Dockerfile`

Use the agent Dockerfile pattern from `references/foundry-agent-patterns.md`.

#### `src/api/agents/hosted_agents.py`

Use the two-mode dispatcher pattern from `references/foundry-agent-patterns.md`.

#### `src/api/routers/chat.py` (Foundry Mode)

```python
"""Chat endpoint — Foundry mode: dispatches to RAG agent instead of calling chain directly."""
from fastapi import APIRouter
from pydantic import BaseModel
from ..agents.hosted_agents import dispatch

router = APIRouter()


class ChatRequest(BaseModel):
    question: str
    session_id: str = "default"
    top_k: int = 5


@router.post("")
async def chat(request: ChatRequest):
    """Send question to the Foundry RAG agent and return structured response."""
    result = await dispatch("rag-agent", {
        "question": request.question,
        "session_id": request.session_id,
        "top_k": request.top_k,
    })
    return result
```

#### `scripts/register_agents.py`

Use the registration script pattern from `references/foundry-agent-patterns.md`.

---

### Chat — Hand-Rolled Mode (R11 = No)

Generate these files ONLY when R11 = No. These use direct Azure OpenAI SDK calls instead of Foundry Agent Service.

### Chat

#### `src/chat/prompts.py`

```python
"""Prompt templates for the RAG chain."""

SYSTEM_PROMPT = """You are a helpful AI assistant that answers questions based on the provided context.

## Instructions
- Answer the user's question using ONLY the information in the context below
- If the context does not contain enough information to answer, say so clearly
- Always cite your sources using [Source N] notation
- Be concise and direct in your responses
- Do not make up information that is not in the context

## Context
{context}
"""

USER_PROMPT = """{question}"""
```

#### `src/chat/chain.py`

```python
"""RAG chain: retrieval + LLM generation."""
import os
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from ..retrieval.retriever import retrieve, format_context
from .prompts import SYSTEM_PROMPT, USER_PROMPT
from .memory import ConversationMemory


def get_chat_client() -> AzureOpenAI:
    token_provider = get_bearer_token_provider(
        DefaultAzureCredential(),
        "https://cognitiveservices.azure.com/.default",
    )
    return AzureOpenAI(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        azure_ad_token_provider=token_provider,
        api_version="2024-10-21",
    )


async def rag_chat(
    question: str,
    session_id: str,
    model: str = "<llm-from-R4>",
    top_k: int = 5,
    stream: bool = True,
):
    """Run the full RAG pipeline: retrieve → build prompt → generate."""
    # 1. Retrieve relevant context
    results = await retrieve(question, top_k=top_k)
    context = format_context(results)

    # 2. Build messages with conversation history
    memory = ConversationMemory.get(session_id)
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT.format(context=context)},
        *memory.get_history(),
        {"role": "user", "content": USER_PROMPT.format(question=question)},
    ]

    # 3. Generate response
    client = get_chat_client()
    if stream:
        response = client.chat.completions.create(
            model=model, messages=messages, stream=True, temperature=0.3,
        )
        # Yield chunks for SSE streaming
        full_response = ""
        for chunk in response:
            if chunk.choices and chunk.choices[0].delta.content:
                content = chunk.choices[0].delta.content
                full_response += content
                yield {"type": "content", "data": content}

        # Save to memory
        memory.add_turn(question, full_response)

        # Return sources
        yield {"type": "sources", "data": [r["source"] for r in results]}
    else:
        response = client.chat.completions.create(
            model=model, messages=messages, stream=False, temperature=0.3,
        )
        answer = response.choices[0].message.content
        memory.add_turn(question, answer)
        yield {
            "type": "complete",
            "data": {"answer": answer, "sources": [r["source"] for r in results]},
        }
```

#### `src/chat/memory.py`

```python
"""Conversation memory — implementation based on R7."""
from collections import defaultdict


class ConversationMemory:
    """In-memory conversation store. Replace with Cosmos DB or Redis for production."""

    _sessions: dict[str, list[dict]] = defaultdict(list)

    @classmethod
    def get(cls, session_id: str) -> "ConversationMemory":
        instance = cls()
        instance.session_id = session_id
        return instance

    def get_history(self, max_turns: int = 10) -> list[dict]:
        """Return the last N turns of conversation history."""
        return self._sessions[self.session_id][-(max_turns * 2):]

    def add_turn(self, question: str, answer: str):
        """Add a question/answer pair to the session history."""
        self._sessions[self.session_id].append({"role": "user", "content": question})
        self._sessions[self.session_id].append({"role": "assistant", "content": answer})

    def clear(self):
        """Clear the session history."""
        self._sessions[self.session_id] = []
```

### API (Hand-Rolled Mode — R11 = No)

Generate this API structure when R11 = No. When R11 = Yes, use the modified `chat.py` router from the Foundry Mode section above instead.

#### `src/api/main.py`

```python
"""FastAPI application for the RAG chatbot."""
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .config import get_settings
from .observability import setup_observability
from .routers import chat, ingest, health


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    setup_observability(settings.app_insights_connection_string)
    # Ensure vector store index exists
    from ..retrieval.vector_store import ensure_index
    ensure_index()
    yield


app = FastAPI(title="RAG Chatbot API", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Restrict in production
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(chat.router, prefix="/chat", tags=["Chat"])
app.include_router(ingest.router, prefix="/ingest", tags=["Ingestion"])
app.include_router(health.router, tags=["Health"])
```

#### `src/api/routers/chat.py`

```python
"""Chat endpoint — supports both streaming (SSE) and synchronous responses."""
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import json
from ...chat.chain import rag_chat

router = APIRouter()


class ChatRequest(BaseModel):
    question: str
    session_id: str = "default"
    stream: bool = True
    top_k: int = 5


@router.post("")
async def chat(request: ChatRequest):
    if request.stream:
        async def event_stream():
            async for chunk in rag_chat(
                question=request.question,
                session_id=request.session_id,
                top_k=request.top_k,
                stream=True,
            ):
                yield f"data: {json.dumps(chunk)}\n\n"
            yield "data: [DONE]\n\n"

        return StreamingResponse(event_stream(), media_type="text/event-stream")
    else:
        result = None
        async for chunk in rag_chat(
            question=request.question,
            session_id=request.session_id,
            top_k=request.top_k,
            stream=False,
        ):
            result = chunk
        return result["data"]
```

---

## Bicep Modules Required

Beyond the base modules (monitoring, container-registry, container-apps-env, container-app, role-assignments):

**Always required:**
- `ai-foundry.bicep` — Azure OpenAI (embedding model from R3 + LLM from R4)

**Based on R2 (vector store):**
- `ai-search.bicep` — if R2 = Azure AI Search (default)
- `cosmos.bicep` — if R2 = Cosmos DB vCore
- (PostgreSQL module if R2 = pgvector)

**Based on R7 (conversation memory) — Hand-Rolled Mode only:**
- `cosmos.bicep` — if R7 = Cosmos DB
- (Redis module if R7 = Redis)

**Based on R1 (data sources):**
- `storage.bicep` — for document storage (almost always needed)

### RBAC for RAG Projects

Container app managed identities need:
- `Cognitive Services OpenAI User` on the AI Services account (for embeddings + chat)
- `Search Index Data Contributor` on Azure AI Search (if R2 = AI Search)
- `Storage Blob Data Reader` on Storage account (for document access)

**Additional RBAC when R11 = Yes (Foundry mode):**
- `Azure AI User` (`53ca9b11-8b9d-4b51-acae-26b3df39f6f0`) on the Foundry account — for both the backend and rag-agent container app managed identities

---

## Type-Specific Quality Checklist

**Both Modes:**
- [ ] Vector store client handles connection failures gracefully
- [ ] Chunking strategy matches R5 answer
- [ ] Embedding dimensions match R3 model (3072 for text-embedding-3-large, 1536 for small/ada)
- [ ] Retrieval chain includes source attribution in responses
- [ ] Ingestion pipeline has idempotency (no duplicate vectors on re-ingestion)
- [ ] Top-k value matches R10 answer
- [ ] All Azure SDK clients use `DefaultAzureCredential` (no API keys in code)
- [ ] Tests include mock vector store and mock LLM for unit testing

**Hand-Rolled Mode (R11 = No) — also verify:**
- [ ] Streaming endpoint uses SSE if R8=yes
- [ ] Conversation memory implementation matches R7 answer
- [ ] Reranking is wired if R9 != none
- [ ] System prompt instructs the LLM to cite sources and avoid hallucination

**Foundry Mode (R11 = Yes) — also verify:**
- [ ] `agent.yaml` description mentions retrieval, grounding, and citations
- [ ] `RAGResponse` schema includes `sources`, `confidence_score`, and `confidence_level`
- [ ] `skill.md` instructs agent to cite sources and avoid hallucination
- [ ] `hosted_agents.py` dispatcher is wired into chat router
- [ ] Ingestion pipeline still works independently (not coupled to agent)
- [ ] `register_agents.py` handles rag-agent registration
- [ ] `docker-compose.yml` includes rag-agent service without `AZURE_AI_PROJECT_ENDPOINT`
