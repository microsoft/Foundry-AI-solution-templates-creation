# Docker Configuration Patterns

Use these patterns when generating Dockerfiles, docker-compose.yml, and related files.

---

## Dockerfile — Python (FastAPI / Flask)

```dockerfile
# --- Build stage ---
FROM python:3.12-slim AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# --- Runtime stage ---
FROM python:3.12-slim

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /install /usr/local

# Copy application code
COPY . .

# Create non-root user
RUN adduser --disabled-password --gecos "" --uid 1001 appuser
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Dockerfile — TypeScript (Next.js)

```dockerfile
# --- Build stage ---
FROM node:20-alpine AS builder

WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build

# --- Runtime stage ---
FROM node:20-alpine

WORKDIR /app

# Copy built assets
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

# Create non-root user
RUN adduser --disabled-password --gecos "" --uid 1001 appuser
USER appuser

EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
```

## Dockerfile — Next.js with Nginx (Static Export)

```dockerfile
# --- Build stage ---
FROM node:20-alpine AS builder

WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build

# --- Runtime stage ---
FROM nginx:alpine

COPY --from=builder /app/out /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

RUN adduser --disabled-password --gecos "" --uid 1001 appuser && \
    chown -R appuser:appuser /var/cache/nginx /var/log/nginx /etc/nginx/conf.d

USER appuser

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## Dockerfile — C# (.NET)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder

WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app
COPY --from=builder /app/publish .

RUN adduser --disabled-password --gecos "" --uid 1001 appuser
USER appuser

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENTRYPOINT ["dotnet", "<ProjectName>.dll"]
```

---

## docker-compose.yml — Template

```yaml
version: "3.8"

services:
  backend:
    build: ./backend           # or ./src depending on project structure
    ports:
      - "8000:8000"
    environment:
      - AZURE_OPENAI_ENDPOINT=${AZURE_OPENAI_ENDPOINT:-}
      - AZURE_OPENAI_DEPLOYMENT_NAME=${AZURE_OPENAI_DEPLOYMENT_NAME:-gpt-4o}
      - APPLICATION_INSIGHTS_CONNECTION_STRING=${APPLICATION_INSIGHTS_CONNECTION_STRING:-}
      # NOTE: Do NOT set AZURE_AI_PROJECT_ENDPOINT here for multi-agent projects
      # Its absence triggers direct container-to-container routing (local mode)
    depends_on:
      # List all dependent services
      - <service-name>
    volumes:
      - ./backend:/app         # Hot-reload for development
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  frontend:                    # Include only if U8=yes
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
    depends_on:
      - backend

  # Add project-type-specific services below
  # For multi-agent: one service per agent on sequential ports
  # For RAG: optional vector store service for local dev
  # For event-driven: message broker service
```

## docker-compose.override.yml — Local Development

```yaml
version: "3.8"

services:
  backend:
    volumes:
      - ./backend:/app
    environment:
      - LOG_LEVEL=DEBUG
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  frontend:
    volumes:
      - ./frontend:/app
      - /app/node_modules       # Exclude node_modules from volume mount
    command: npm run dev
```

---

## .dockerignore

Generate a `.dockerignore` file in each service directory:

```
__pycache__
*.pyc
*.pyo
.git
.gitignore
.env
.env.*
*.md
.vscode
.idea
node_modules
.next
dist
coverage
.pytest_cache
.mypy_cache
```

---

## nginx.conf (for frontend static hosting)

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback — all routes serve index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy (for local dev without CORS issues)
    location /api/ {
        proxy_pass http://backend:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Port Conventions

| Service | Default Port | Notes |
|---------|-------------|-------|
| Backend API | 8000 | Always the primary API port |
| Frontend | 3000 | Next.js dev server or Nginx |
| Agent 1 | 8001 | Multi-agent projects |
| Agent 2 | 8002 | Multi-agent projects |
| Agent 3 | 8003 | Multi-agent projects |
| Agent N | 800N | Sequential ports per agent |

---

## Docker Best Practices

1. **Multi-stage builds**: Always use multi-stage to minimize image size
2. **Non-root user**: Create and switch to a non-root user in every Dockerfile
3. **Health checks**: Include HEALTHCHECK in every Dockerfile
4. **No secrets in images**: Never COPY .env files or embed secrets
5. **Layer caching**: Copy dependency files first, install dependencies, then copy source
6. **Timestamp tags**: Use `YYYYMMDDHHmmss` tags, never `latest`
7. **Platform**: Build for `linux/amd64` when targeting Azure Container Apps
