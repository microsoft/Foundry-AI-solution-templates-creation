# CI/CD Pipeline Patterns — GitHub Actions

Use these templates when generating `.github/workflows/ci.yml` and `.github/workflows/deploy.yml`.

---

## `ci.yml` — Pull Request Validation

This workflow runs on every PR to validate code quality, tests, and infrastructure.

### Python Project

```yaml
name: CI

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install ruff mypy pytest pytest-asyncio
          pip install -r requirements.txt
          # Install additional service requirements if they exist
          for req in */requirements.txt; do
            pip install -r "$req" 2>/dev/null || true
          done

      - name: Lint with ruff
        run: ruff check .

      - name: Type check with mypy
        run: mypy . --ignore-missing-imports

      - name: Run tests
        run: pytest tests/ -v --tb=short

  validate-bicep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate Bicep templates
        run: az bicep build --file infra/main.bicep

  validate-docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate docker-compose
        run: docker compose config --quiet
```

### TypeScript Project (or with frontend)

Add this job to the CI workflow when the project includes TypeScript:

```yaml
  lint-and-test-frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./frontend
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npx tsc --noEmit

      - name: Run tests
        run: npm test -- --passWithNoTests
```

### C# Project

```yaml
  build-and-test-dotnet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "8.0.x"

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore

      - name: Test
        run: dotnet test --no-build --verbosity normal
```

---

## `deploy.yml` — Deploy on Push to Main

This workflow runs when code is pushed to main. It builds Docker images, pushes to ACR, and triggers deployment.

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

env:
  AZURE_ENV_NAME: ${{ vars.AZURE_ENV_NAME }}
  AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Install azd
        uses: Azure/setup-azd@v1

      - name: Log in with Azure (federated credentials)
        run: |
          azd auth login \
            --client-id "${{ vars.AZURE_CLIENT_ID }}" \
            --federated-credential-provider "github" \
            --tenant-id "${{ vars.AZURE_TENANT_ID }}"

      - name: Provision and deploy
        run: azd up --no-prompt
        env:
          AZURE_ENV_NAME: ${{ vars.AZURE_ENV_NAME }}
          AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}
          AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Run smoke test
        run: |
          # Get the deployed backend URL
          BACKEND_URL=$(azd env get-value BACKEND_URL 2>/dev/null || echo "")
          if [[ -n "$BACKEND_URL" ]]; then
            echo "Running smoke test against $BACKEND_URL"
            curl -sf "$BACKEND_URL/health" || exit 1
            echo "Smoke test passed"
          else
            echo "WARNING: BACKEND_URL not available, skipping smoke test"
          fi
```

---

## Workflow Customization by Project Type

| Project Type | CI Additions | Deploy Additions |
|---|---|---|
| RAG Chatbot | Test ingestion pipeline, test retrieval chain | Run index creation after deploy |
| Multi-Agent | Test each agent schema, test orchestrator logic | Run `register_agents.py` after deploy |
| API Backend | Test API endpoints, test database migrations | Run database migrations after deploy |
| Data Pipeline | Test transformers, validate pipeline definitions | Deploy pipeline definitions |
| Azure Functions | Test function triggers, validate bindings | Deploy function app via `func azure functionapp publish` |
| Full-Stack Web App | Test frontend + backend, E2E tests (optional) | Build and deploy both containers |
| ML Training | Test training scripts with small dataset | No auto-deploy (training is manual) |
| Event-Driven | Test message handlers, test event schemas | Create queues/topics after deploy |

---

## CI/CD Best Practices

1. **Separate CI and Deploy**: CI runs on PRs (fast feedback), Deploy runs on main merge
2. **Cache dependencies**: Use `cache` options in setup actions for faster runs
3. **Parallel jobs**: Run lint, test, and Bicep validation in parallel
4. **Fail fast**: CI should fail quickly on the first error
5. **Smoke tests**: Always include a basic health check after deployment
6. **Federated credentials**: Use `id-token: write` permission for keyless Azure auth
7. **Environment protection**: Use GitHub environments with required reviewers for production
