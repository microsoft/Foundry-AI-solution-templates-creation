# azure.yaml and Deployment Hook Patterns

Use these templates when generating `azure.yaml` and the associated hook scripts.

---

## `azure.yaml` — Template

```yaml
name: <project-slug>
metadata:
  template: <project-slug>@1.0
requiredVersions:
  azd: '>= 1.18.0'

hooks:
  preprovision:
    # Validate region is supported for the chosen model/services
    # Prompt user for configuration choices if needed
    # Exit non-zero on validation failure to block broken deploys
    windows:
      run: scripts/preprovision.ps1
      shell: pwsh
    posix:
      run: scripts/preprovision.sh
      shell: sh

  postprovision:
    # 1. az acr login
    # 2. Build and push Docker images with timestamp tag
    # 3. Run registration/migration scripts if needed
    windows:
      run: scripts/postprovision.ps1
      shell: pwsh
    posix:
      run: scripts/postprovision.sh
      shell: sh
```

---

## `azure.yaml` Conventions

- `IMAGE_TAG` must always be a timestamp (`YYYYMMDDHHmmss`) — never use `latest` (prevents stale image issues)
- `preprovision` must exit non-zero on validation failure to block broken deploys
- `postprovision` must validate all images exist in ACR before running registration scripts
- All hook scripts must work on both Windows (PowerShell) and Linux/macOS (bash)

---

## Hook Scripts

### `scripts/preprovision.sh` (Posix)

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Pre-provision validation ==="

# Validate required environment variables
: "${AZURE_ENV_NAME:?AZURE_ENV_NAME is required}"
: "${AZURE_LOCATION:?AZURE_LOCATION is required}"

# Validate region (customize based on project type)
# For AI projects, validate model availability in the selected region
VALID_REGIONS=("eastus2" "swedencentral" "westus3" "eastus")

REGION_VALID=false
for region in "${VALID_REGIONS[@]}"; do
    if [[ "$AZURE_LOCATION" == "$region" ]]; then
        REGION_VALID=true
        break
    fi
done

if [[ "$REGION_VALID" != "true" ]]; then
    echo "ERROR: Region '$AZURE_LOCATION' is not supported."
    echo "Supported regions: ${VALID_REGIONS[*]}"
    echo ""
    echo "To fix: run 'azd env set AZURE_LOCATION <supported-region>'"
    exit 1
fi

echo "Region validation passed: $AZURE_LOCATION"
echo "=== Pre-provision validation complete ==="
```

### `scripts/preprovision.ps1` (Windows)

```powershell
$ErrorActionPreference = "Stop"

Write-Host "=== Pre-provision validation ==="

# Validate required environment variables
if (-not $env:AZURE_ENV_NAME) { throw "AZURE_ENV_NAME is required" }
if (-not $env:AZURE_LOCATION) { throw "AZURE_LOCATION is required" }

# Validate region
$validRegions = @("eastus2", "swedencentral", "westus3", "eastus")

if ($env:AZURE_LOCATION -notin $validRegions) {
    Write-Error "Region '$($env:AZURE_LOCATION)' is not supported."
    Write-Error "Supported regions: $($validRegions -join ', ')"
    Write-Error ""
    Write-Error "To fix: run 'azd env set AZURE_LOCATION <supported-region>'"
    exit 1
}

Write-Host "Region validation passed: $env:AZURE_LOCATION"
Write-Host "=== Pre-provision validation complete ==="
```

### `scripts/postprovision.sh` (Posix)

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Post-provision: Build and deploy ==="

# Get outputs from azd provisioning
ACR_NAME=$(azd env get-value ACR_NAME 2>/dev/null || echo "")
if [[ -z "$ACR_NAME" ]]; then
    echo "ERROR: ACR_NAME not found. Provisioning may have failed."
    exit 1
fi

# Generate timestamp-based image tag (never use 'latest')
IMAGE_TAG=$(date +%Y%m%d%H%M%S)
echo "Image tag: $IMAGE_TAG"

# Login to ACR
echo "Logging in to ACR: $ACR_NAME"
az acr login --name "$ACR_NAME"

# Build and push images
# Customize this list based on the project's services
SERVICES=("backend")
# Add more services based on project type:
# For multi-agent: add each agent name
# For full-stack: add "frontend"
# For Foundry-enabled (U11=Yes): add each agent directory name from agents/

# If U11=Yes, discover and add agent services from agents/ directory
if [[ -d "./agents" ]]; then
    for AGENT_DIR in ./agents/*/; do
        AGENT_NAME=$(basename "$AGENT_DIR")
        SERVICES+=("agents/${AGENT_NAME}")
    done
fi

for SERVICE in "${SERVICES[@]}"; do
    IMAGE_NAME=$(basename "$SERVICE")
    echo "Building image: $IMAGE_NAME"
    az acr build \
        --registry "$ACR_NAME" \
        --image "${IMAGE_NAME}:${IMAGE_TAG}" \
        --platform linux/amd64 \
        "./${SERVICE}"
done

# Verify all images exist
echo "Verifying images in ACR..."
for SERVICE in "${SERVICES[@]}"; do
    IMAGE_NAME=$(basename "$SERVICE")
    az acr repository show-tags --name "$ACR_NAME" --repository "$IMAGE_NAME" --query "[?contains(@, '$IMAGE_TAG')]" -o tsv
    if [[ $? -ne 0 ]]; then
        echo "ERROR: Image ${IMAGE_NAME}:${IMAGE_TAG} not found in ACR"
        exit 1
    fi
done

# Run post-deploy scripts
# Customize based on project type

# Register Foundry agents (U11=Yes or Multi-Agent projects)
# Only run if register_agents.py exists (indicates Foundry agent project)
if [[ -f "scripts/register_agents.py" ]]; then
    echo "Registering Foundry agents..."
    AZURE_AI_PROJECT_ENDPOINT=$(azd env get-value AZURE_AI_PROJECT_ENDPOINT 2>/dev/null || echo "")
    if [[ -n "$AZURE_AI_PROJECT_ENDPOINT" ]]; then
        python scripts/register_agents.py
    else
        echo "WARNING: AZURE_AI_PROJECT_ENDPOINT not set, skipping agent registration"
    fi
fi

# Database migrations (API Backend, Full-Stack)
# python scripts/migrate.py

echo "=== Post-provision complete ==="
```

### `scripts/postprovision.ps1` (Windows)

```powershell
$ErrorActionPreference = "Stop"

Write-Host "=== Post-provision: Build and deploy ==="

# Get outputs from azd provisioning
$ACR_NAME = azd env get-value ACR_NAME 2>$null
if (-not $ACR_NAME) {
    throw "ACR_NAME not found. Provisioning may have failed."
}

# Generate timestamp-based image tag
$IMAGE_TAG = Get-Date -Format "yyyyMMddHHmmss"
Write-Host "Image tag: $IMAGE_TAG"

# Login to ACR
Write-Host "Logging in to ACR: $ACR_NAME"
az acr login --name $ACR_NAME

# Build and push images
$services = @("backend")
# Add more services based on project type
# For Foundry-enabled (U11=Yes): add each agent directory name from agents/

# If U11=Yes, discover and add agent services from agents/ directory
if (Test-Path "./agents") {
    Get-ChildItem -Path "./agents" -Directory | ForEach-Object {
        $services += "agents/$($_.Name)"
    }
}

foreach ($service in $services) {
    $imageName = Split-Path $service -Leaf
    Write-Host "Building image: $imageName"
    az acr build `
        --registry $ACR_NAME `
        --image "${imageName}:${IMAGE_TAG}" `
        --platform linux/amd64 `
        "./$service"
}

# Verify all images exist
Write-Host "Verifying images in ACR..."
foreach ($service in $services) {
    $imageName = Split-Path $service -Leaf
    $tags = az acr repository show-tags --name $ACR_NAME --repository $imageName --query "[?contains(@, '$IMAGE_TAG')]" -o tsv
    if (-not $tags) {
        throw "Image ${imageName}:${IMAGE_TAG} not found in ACR"
    }
}

# Run post-deploy scripts

# Register Foundry agents (U11=Yes or Multi-Agent projects)
# Only run if register_agents.py exists (indicates Foundry agent project)
if (Test-Path "scripts/register_agents.py") {
    Write-Host "Registering Foundry agents..."
    $projectEndpoint = azd env get-value AZURE_AI_PROJECT_ENDPOINT 2>$null
    if ($projectEndpoint) {
        python scripts/register_agents.py
    } else {
        Write-Warning "AZURE_AI_PROJECT_ENDPOINT not set, skipping agent registration"
    }
}

# Database migrations (API Backend, Full-Stack)
# python scripts/migrate.py

Write-Host "=== Post-provision complete ==="
```

---

## Customization by Project Type

| Project Type | preprovision additions | postprovision additions |
|---|---|---|
| RAG Chatbot (U11=No) | Validate AI model region, validate search service region | Build API container, run index creation |
| RAG Chatbot (U11=Yes) | Validate AI model region, validate search service region, validate Foundry compatibility | Build API + agent containers, run index creation, run `register_agents.py` |
| Multi-Agent | Validate AI model region, validate Foundry compatibility | Build agent containers + backend, run `register_agents.py` |
| API Backend | Validate database region | Build API container, run database migrations |
| API Backend (A9=Yes, U11=Yes) | Validate database region, validate AI model region, validate Foundry compatibility | Build API + agent containers, run database migrations, run `register_agents.py` |
| Data Pipeline | Validate data service availability | Deploy pipeline definitions |
| Data Pipeline (D9=Yes, U11=Yes) | Validate data service availability, validate AI model region, validate Foundry compatibility | Deploy pipeline definitions, build agent container, run `register_agents.py` |
| Azure Functions | Validate Functions runtime availability | Deploy function app |
| Azure Functions (F7=Yes, U11=Yes) | Validate Functions runtime availability, validate AI model region, validate Foundry compatibility | Deploy function app, build agent container, run `register_agents.py` |
| Full-Stack Web App | Validate all service regions | Build frontend + backend containers |
| Full-Stack Web App (W10=Yes, U11=Yes) | Validate all service regions, validate AI model region, validate Foundry compatibility | Build frontend + backend + agent containers, run `register_agents.py` |
| ML Training | Validate ML compute region, validate GPU availability | Build training container, create ML compute |
| Event-Driven | Validate Service Bus region | Build service containers, create queues/topics |
| Event-Driven (E9=Yes, U11=Yes) | Validate Service Bus region, validate AI model region, validate Foundry compatibility | Build service + agent containers, create queues/topics, run `register_agents.py` |
