# Full-Stack Web App — Scaffold Pattern

This reference file defines the scaffold pattern for **full-stack web applications** with frontend UI and backend API deployed on Azure Container Apps.

---

## Type-Specific Questions

| # | Question | Guidance |
|---|---|---|
| W1 | **Frontend framework?** | `Next.js` (default, React-based, SSR), `React SPA` (client-only), `Vue.js`, `Angular`. |
| W2 | **Rendering strategy?** | `SSR` (default, server-side rendering), `SSG` (static generation), `SPA` (client-side only). |
| W3 | **Backend framework?** | `FastAPI` (default for Python), `Express` (TypeScript), `ASP.NET` (C#). |
| W4 | **Database?** | `Cosmos DB` (default), `PostgreSQL Flexible Server`, `MongoDB`. |
| W5 | **Key pages/routes?** | List the main pages (e.g., Dashboard, Settings, Profile, Detail view). Drives page structure. |
| W6 | **UI component library?** | `shadcn/ui` (default for React/Next.js), `Material UI`, `Ant Design`, `None` (custom styling). |
| W7 | **State management?** | `React Context` (default, simple), `Zustand` (lightweight), `Redux` (complex). |
| W8 | **Real-time features?** | `None` (default), `Polling`, `Server-Sent Events`, `WebSockets`. |
| W9 | **File uploads?** | `None` (default), `Azure Blob Storage` (direct upload), `Backend proxy upload`. |
| W10 | **Does this app include AI features?** | `No` (default). If `Yes`: chat, summarization, content generation, etc. Triggers U11 question. |
| W11 | **What AI features?** | Only ask if W10=Yes. Options: `Chat/Q&A`, `Summarization`, `Content generation`, `Data analysis`, `Custom` (describe). Drives agent skill and frontend AI components. |

---

## Project Folder Structure

```
<project-slug>/
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt            # or package.json for Express
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                 # App factory + lifespan
│   │   ├── config.py               # pydantic-settings
│   │   ├── observability.py        # OTel setup
│   │   ├── routers/
│   │   │   ├── __init__.py
│   │   │   ├── <entity>.py         # API endpoint per entity
│   │   │   ├── auth.py             # Authentication endpoints
│   │   │   └── health.py           # Health check
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   └── schemas.py          # Shared Pydantic schemas
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   └── <entity>_service.py
│   │   └── db/
│   │       ├── __init__.py
│   │       └── client.py           # Database client (from W4)
│   └── tests/
│       ├── __init__.py
│       └── test_api.py
│
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf                  # For production static serving
│   ├── next.config.ts              # or vite.config.ts
│   ├── package.json
│   ├── tsconfig.json
│   ├── postcss.config.mjs
│   ├── tailwind.config.ts
│   ├── components.json             # shadcn/ui config (if W6=shadcn/ui)
│   │
│   ├── app/                        # Next.js App Router (or pages/ for Pages Router)
│   │   ├── globals.css
│   │   ├── layout.tsx              # Root layout with providers
│   │   ├── page.tsx                # Home page
│   │   └── <route>/
│   │       └── page.tsx            # One page per route from W5
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── header.tsx
│   │   │   ├── sidebar.tsx
│   │   │   └── footer.tsx
│   │   ├── <feature>/              # Feature-specific components
│   │   │   └── <component>.tsx
│   │   └── ui/                     # Shared UI primitives (shadcn/ui)
│   │
│   └── lib/
│       ├── api.ts                  # Typed API client
│       ├── types.ts                # TypeScript interfaces mirroring backend schemas
│       ├── hooks/                  # Custom React hooks
│       │   └── use-<feature>.ts
│       └── utils.ts
│
└── shared/                         # Optional: shared type definitions
    └── types.ts                    # Types used by both frontend and backend
```

---

## Source File Patterns

### Frontend — API Client

```typescript
// frontend/lib/api.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000";

async function fetchAPI<T>(path: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...options?.headers,
    },
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ detail: "Unknown error" }));
    throw new Error(error.detail || `HTTP ${response.status}`);
  }

  return response.json();
}

export const api = {
  get: <T>(path: string) => fetchAPI<T>(path),
  post: <T>(path: string, data: unknown) =>
    fetchAPI<T>(path, { method: "POST", body: JSON.stringify(data) }),
  put: <T>(path: string, data: unknown) =>
    fetchAPI<T>(path, { method: "PUT", body: JSON.stringify(data) }),
  delete: (path: string) => fetchAPI(path, { method: "DELETE" }),
};
```

### Frontend — Root Layout

```typescript
// frontend/app/layout.tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "<Project Name>",
  description: "<From U1>",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <div className="min-h-screen flex flex-col">
          {/* Header */}
          <header className="border-b px-6 py-4">
            <h1 className="text-xl font-semibold"><Project Name></h1>
          </header>

          {/* Main content */}
          <main className="flex-1 p-6">{children}</main>
        </div>
      </body>
    </html>
  );
}
```

---

## AI Features Layer (W10 = Yes, U11 = Yes)

When the app includes AI features powered by Foundry Agent Service, add these files. Read `references/foundry-agent-patterns.md` for the shared patterns referenced below.

### Additional Backend Files

```
<project-slug>/
├── agents/
│   └── <ai-task>-agent/               # e.g., chat-agent, summarizer-agent
│       ├── agent.yaml
│       ├── Dockerfile
│       ├── main.py
│       ├── requirements.txt
│       ├── schemas.py
│       └── skills/
│           └── <ai-task>-skill/
│               └── skill.md
├── backend/app/
│   ├── agents/
│   │   ├── __init__.py
│   │   └── hosted_agents.py            # Two-mode dispatcher
│   └── routers/
│       └── ai.py                       # AI endpoints
├── scripts/
│   └── register_agents.py
```

### AI Backend Router

```python
# backend/app/routers/ai.py
"""AI feature endpoints — powered by Foundry Agent Service."""
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from ..agents.hosted_agents import dispatch
import json

router = APIRouter(prefix="/ai", tags=["AI"])


class ChatMessage(BaseModel):
    message: str
    session_id: str = "default"


class SummarizeRequest(BaseModel):
    text: str
    max_length: int = 200


@router.post("/chat")
async def ai_chat(request: ChatMessage):
    """Chat endpoint powered by Foundry agent."""
    result = await dispatch("<ai-task>-agent", {
        "message": request.message,
        "session_id": request.session_id,
    })
    return result


@router.post("/summarize")
async def ai_summarize(request: SummarizeRequest):
    """Summarization endpoint powered by Foundry agent."""
    result = await dispatch("summarizer-agent", {
        "text": request.text,
        "max_length": request.max_length,
    })
    return result
```

Choose endpoints based on W11 answer. Not all endpoints are needed — only those matching the selected AI features.

### Additional Frontend Components

```
frontend/
├── components/
│   └── ai/
│       ├── chat-interface.tsx          # Chat widget with streaming
│       ├── ai-result-panel.tsx         # Display structured AI results
│       └── summarize-button.tsx        # Inline summarization trigger
└── lib/
    └── ai-api.ts                      # SSE-capable API client for AI endpoints
```

#### `frontend/lib/ai-api.ts`

```typescript
const API_BASE = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000";

export async function aiChat(message: string, sessionId: string = "default") {
  const response = await fetch(`${API_BASE}/ai/chat`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message, session_id: sessionId }),
  });
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}

export async function aiSummarize(text: string, maxLength: number = 200) {
  const response = await fetch(`${API_BASE}/ai/summarize`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text, max_length: maxLength }),
  });
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}
```

#### `frontend/components/ai/chat-interface.tsx`

```tsx
"use client";
import { useState } from "react";
import { aiChat } from "@/lib/ai-api";

interface Message {
  role: "user" | "assistant";
  content: string;
}

export function ChatInterface() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);

  async function handleSend() {
    if (!input.trim()) return;
    const userMsg: Message = { role: "user", content: input };
    setMessages((prev) => [...prev, userMsg]);
    setInput("");
    setLoading(true);

    try {
      const result = await aiChat(input);
      setMessages((prev) => [
        ...prev,
        { role: "assistant", content: result.summary || result.answer },
      ]);
    } catch {
      setMessages((prev) => [
        ...prev,
        { role: "assistant", content: "Sorry, something went wrong." },
      ]);
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="flex flex-col h-full">
      <div className="flex-1 overflow-y-auto space-y-4 p-4">
        {messages.map((msg, i) => (
          <div key={i} className={msg.role === "user" ? "text-right" : "text-left"}>
            <span className="inline-block px-4 py-2 rounded-lg bg-gray-100">
              {msg.content}
            </span>
          </div>
        ))}
        {loading && <div className="text-gray-400">Thinking...</div>}
      </div>
      <div className="border-t p-4 flex gap-2">
        <input
          className="flex-1 border rounded px-3 py-2"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && handleSend()}
          placeholder="Ask a question..."
        />
        <button
          className="px-4 py-2 bg-blue-600 text-white rounded"
          onClick={handleSend}
          disabled={loading}
        >
          Send
        </button>
      </div>
    </div>
  );
}
```

### AI Mode — Hand-Rolled (W10 = Yes, U11 = No)

If the user declines Foundry, add `backend/app/services/ai_service.py` using direct Azure OpenAI SDK (same pattern as in `references/api-backend.md` hand-rolled section). Frontend components remain the same — they call the same HTTP endpoints.

---

## Bicep Modules Required

- `container-apps-env.bicep` + `container-app.bicep` (for backend + frontend)
- `container-registry.bicep` (always)
- `monitoring.bicep` (always)
- `cosmos.bicep` or PostgreSQL — based on W4
- `storage.bicep` — if W9 includes file uploads

**If W10 = Yes and U11 = Yes (Foundry AI mode):**
- `ai-foundry.bicep` — Foundry account + project + model deployment
- Additional `container-app.bicep` instance for agent container(s)
- RBAC: `Cognitive Services OpenAI User` + `Azure AI User` on Foundry account

**If W10 = Yes and U11 = No (Hand-rolled AI mode):**
- `ai-foundry.bicep` — Azure OpenAI model deployment only

---

## Type-Specific Quality Checklist

- [ ] Frontend `lib/types.ts` mirrors all backend Pydantic schemas
- [ ] API client handles errors and displays user-friendly messages
- [ ] All routes from W5 have corresponding pages
- [ ] Layout includes consistent header/navigation across all pages
- [ ] Frontend uses `NEXT_PUBLIC_API_URL` env var for backend URL
- [ ] Backend CORS middleware allows frontend origin
- [ ] Responsive design works on mobile and desktop
- [ ] Loading states shown during API calls
- [ ] Error boundaries catch and display component errors
- [ ] State management matches W7 answer
- [ ] Real-time features match W8 answer if applicable
- [ ] File upload implementation matches W9 answer if applicable

**If W10 = Yes and U11 = Yes (Foundry AI mode) — also verify:**
- [ ] AI router registered in backend main.py
- [ ] Frontend AI components (chat-interface, ai-result-panel) are integrated into page layout
- [ ] `ai-api.ts` client handles AI endpoint responses correctly
- [ ] Run Foundry quality checklist from `references/foundry-agent-patterns.md`
