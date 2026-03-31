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

## Bicep Modules Required

- `container-apps-env.bicep` + `container-app.bicep` (for backend + frontend)
- `container-registry.bicep` (always)
- `monitoring.bicep` (always)
- `cosmos.bicep` or PostgreSQL — based on W4
- `storage.bicep` — if W9 includes file uploads

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
