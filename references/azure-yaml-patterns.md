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

for SERVICE in "${SERVICES[@]}"; do
    echo "Building image: $SERVICE"
    az acr build \
        --registry "$ACR_NAME" \
        --image "${SERVICE}:${IMAGE_TAG}" \
        --platform linux/amd64 \
        "./${SERVICE}"
done

# Verify all images exist
echo "Verifying images in ACR..."
for SERVICE in "${SERVICES[@]}"; do
    az acr repository show-tags --name "$ACR_NAME" --repository "$SERVICE" --query "[?contains(@, '$IMAGE_TAG')]" -o tsv
    if [[ $? -ne 0 ]]; then
        echo "ERROR: Image ${SERVICE}:${IMAGE_TAG} not found in ACR"
        exit 1
    fi
done

# Run post-deploy scripts (e.g., agent registration, database migrations)
# Customize based on project type
# python scripts/register_agents.py  # (for multi-agent projects)

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

foreach ($service in $services) {
    Write-Host "Building image: $service"
    az acr build `
        --registry $ACR_NAME `
        --image "${service}:${IMAGE_TAG}" `
        --platform linux/amd64 `
        "./$service"
}

# Verify all images exist
Write-Host "Verifying images in ACR..."
foreach ($service in $services) {
    $tags = az acr repository show-tags --name $ACR_NAME --repository $service --query "[?contains(@, '$IMAGE_TAG')]" -o tsv
    if (-not $tags) {
        throw "Image ${service}:${IMAGE_TAG} not found in ACR"
    }
}

Write-Host "=== Post-provision complete ==="
```

---

## Customization by Project Type

| Project Type | preprovision additions | postprovision additions |
|---|---|---|
| RAG Chatbot | Validate AI model region, validate search service region | Build API container, run index creation |
| Multi-Agent | Validate AI model region, validate Foundry compatibility | Build agent containers + backend, run `register_agents.py` |
| API Backend | Validate database region | Build API container, run database migrations |
| Data Pipeline | Validate data service availability | Deploy pipeline definitions |
| Azure Functions | Validate Functions runtime availability | Deploy function app |
| Full-Stack Web App | Validate all service regions | Build frontend + backend containers |
| ML Training | Validate ML compute region, validate GPU availability | Build training container, create ML compute |
| Event-Driven | Validate Service Bus region | Build service containers, create queues/topics |
