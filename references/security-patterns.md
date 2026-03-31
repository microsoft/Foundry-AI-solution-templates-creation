# Security Patterns — Auth, RBAC, Key Vault, Network Isolation

These patterns apply to production hardening. The default scaffold ships without auth for simplicity. Document these as migration steps in `docs/production-migration.md`.

---

## Authentication — Microsoft Entra ID

### App Registration

```
1. Register an application in Microsoft Entra ID
2. Configure redirect URIs for the frontend (e.g., https://<frontend-url>/auth/callback)
3. Create a client secret or configure certificate authentication
4. Set required API permissions
5. Configure token validation in the backend
```

### FastAPI Middleware

```python
# middleware/auth.py
from fastapi import Request, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from functools import lru_cache

security = HTTPBearer()

TENANT_ID = os.environ["AZURE_TENANT_ID"]
CLIENT_ID = os.environ["AZURE_CLIENT_ID"]
AUTHORITY = f"https://login.microsoftonline.com/{TENANT_ID}"
JWKS_URL = f"{AUTHORITY}/discovery/v2.0/keys"


async def validate_token(credentials: HTTPAuthorizationCredentials) -> dict:
    """Validate a JWT token from Microsoft Entra ID."""
    token = credentials.credentials
    try:
        # Decode and validate the token
        # Use python-jose or PyJWT with jwks-client for production
        payload = jwt.decode(
            token,
            options={"verify_signature": True},
            audience=CLIENT_ID,
            issuer=f"{AUTHORITY}/v2.0",
        )
        return payload
    except jwt.PyJWTError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {e}")
```

### Frontend Auth (Next.js with MSAL)

```typescript
// lib/auth.ts
import { PublicClientApplication } from "@azure/msal-browser";

const msalConfig = {
  auth: {
    clientId: process.env.NEXT_PUBLIC_AZURE_CLIENT_ID!,
    authority: `https://login.microsoftonline.com/${process.env.NEXT_PUBLIC_AZURE_TENANT_ID}`,
    redirectUri: window.location.origin,
  },
};

export const msalInstance = new PublicClientApplication(msalConfig);

export async function getAccessToken(): Promise<string> {
  const accounts = msalInstance.getAllAccounts();
  if (accounts.length === 0) {
    await msalInstance.loginPopup({ scopes: ["api://<client-id>/access"] });
  }
  const response = await msalInstance.acquireTokenSilent({
    scopes: ["api://<client-id>/access"],
    account: msalInstance.getAllAccounts()[0],
  });
  return response.accessToken;
}
```

---

## RBAC — Role-Based Access Control

### Role Definitions

Define roles based on U2 (end users):

```python
# models/roles.py
from enum import Enum

class AppRole(str, Enum):
    ADMIN = "Admin"          # Full access: manage users, configure agents, view all data
    REVIEWER = "Reviewer"    # Submit and review cases, override decisions
    VIEWER = "Viewer"        # Read-only access to results and reports
```

### Role Check Decorator

```python
from functools import wraps
from fastapi import Depends, HTTPException

def require_role(*roles: AppRole):
    """FastAPI dependency that checks the user's role."""
    async def check(token: dict = Depends(validate_token)):
        user_roles = token.get("roles", [])
        if not any(role.value in user_roles for role in roles):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return token
    return check

# Usage:
@router.post("/admin/settings")
async def update_settings(user=Depends(require_role(AppRole.ADMIN))):
    ...
```

---

## Azure Key Vault Integration

### Storing Secrets

```python
# services/keyvault.py
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

def get_secret(vault_url: str, secret_name: str) -> str:
    """Retrieve a secret from Azure Key Vault."""
    client = SecretClient(vault_url=vault_url, credential=DefaultAzureCredential())
    return client.get_secret(secret_name).value
```

### ACA Secret Store Integration

In `container-app.bicep`, reference Key Vault secrets:

```bicep
properties: {
  configuration: {
    secrets: [
      {
        name: 'my-secret'
        keyVaultUrl: '${keyVaultUri}secrets/my-secret'
        identity: 'system'
      }
    ]
  }
}
```

---

## Network Isolation

### VNet Injection for ACA

```bicep
// container-apps-env.bicep — with VNet
param subnetId string

resource env 'Microsoft.App/managedEnvironments@2024-03-01' = {
  properties: {
    vnetConfiguration: {
      infrastructureSubnetId: subnetId
      internal: true   // No public access — use private endpoints
    }
  }
}
```

### Private Endpoints for AI Services

```bicep
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-${foundryAccountName}'
  location: location
  properties: {
    subnet: { id: subnetId }
    privateLinkServiceConnections: [
      {
        name: 'plsc-${foundryAccountName}'
        properties: {
          privateLinkServiceId: foundryAccount.id
          groupIds: ['account']
        }
      }
    ]
  }
}
```

---

## Input Validation and Sanitization

```python
# middleware/validation.py
from fastapi import Request, HTTPException

MAX_PAYLOAD_SIZE = 10 * 1024 * 1024  # 10MB

async def validate_payload_size(request: Request):
    """Reject oversized payloads."""
    content_length = request.headers.get("content-length")
    if content_length and int(content_length) > MAX_PAYLOAD_SIZE:
        raise HTTPException(status_code=413, detail="Payload too large")
```

---

## Production Migration Checklist

Document these items in `docs/production-migration.md`:

| Area | Default Scaffold | Production Requirement |
|------|-----------------|----------------------|
| **Authentication** | None | Microsoft Entra ID + app registration |
| **RBAC** | None | Role-based access aligned with user types (U2) |
| **Secrets** | Env vars in ACA | Azure Key Vault with secret store reference |
| **Network** | Public endpoints | VNet injection + private endpoints |
| **Container security** | Base image | Non-root user, vulnerability scanning in ACR |
| **API rate limiting** | None | Azure API Management or FastAPI middleware |
| **Input validation** | Pydantic only | Sanitize all inputs, reject oversized payloads |
| **Storage** | In-memory | Azure Cosmos DB or PostgreSQL |
| **Session isolation** | Single process | Distributed sessions with locking |
| **Scaling** | Default ACA | HTTP-based autoscale with min replicas |
| **Async processing** | Sync HTTP | Azure Service Bus for long-running tasks |
| **Logging** | Console | Structured JSON logging to Azure Monitor |
| **Backup & DR** | None | Cosmos DB continuous backup, geo-redundant storage |
