# Observability Patterns — OpenTelemetry + Azure Monitor

Use these patterns when setting up observability in scaffolded projects.

---

## Environment Variable Normalization

The Microsoft Agent Framework (MAF) adapter reads the **no-underscore** variant of the Application Insights connection string. Apply this normalization in every entry point:

```python
import os

_ai_conn = os.environ.get("APPLICATION_INSIGHTS_CONNECTION_STRING")
if _ai_conn:
    os.environ.setdefault("APPLICATIONINSIGHTS_CONNECTION_STRING", _ai_conn)
```

This must appear in:
- Every agent `main.py` (for multi-agent projects)
- The backend `app/main.py` or `src/api/main.py`

---

## Backend Observability Setup

### Python (FastAPI)

```python
# observability.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.semconv.resource import ResourceAttributes
import logging


def setup_observability(connection_string: str | None, service_name: str = "backend") -> None:
    """Configure OpenTelemetry with Azure Monitor exporter."""
    if not connection_string:
        logging.info("No Application Insights connection string — observability disabled")
        return

    from azure.monitor.opentelemetry.exporter import AzureMonitorTraceExporter

    resource = Resource.create({
        ResourceAttributes.SERVICE_NAME: service_name,
    })

    provider = TracerProvider(resource=resource)
    exporter = AzureMonitorTraceExporter(connection_string=connection_string)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    logging.info(f"Observability configured for service: {service_name}")
```

### Calling from FastAPI lifespan

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from .config import get_settings
from .observability import setup_observability


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    # Normalize env var for MAF compatibility
    import os
    _ai_conn = os.environ.get("APPLICATION_INSIGHTS_CONNECTION_STRING")
    if _ai_conn:
        os.environ.setdefault("APPLICATIONINSIGHTS_CONNECTION_STRING", _ai_conn)

    setup_observability(settings.app_insights_connection_string, service_name="backend")
    yield


app = FastAPI(lifespan=lifespan)
```

---

## Agent Observability (Multi-Agent Projects)

For Foundry Hosted Agents, `from_agent_framework().run()` **auto-configures OpenTelemetry** when the `APPLICATIONINSIGHTS_CONNECTION_STRING` environment variable is set. No manual OTel setup needed in agent containers — just the env var normalization.

Each agent span automatically includes:
- `gen_ai.agent.name` — for Foundry Traces correlation
- `gen_ai.system` — identifies the AI provider
- Request/response token counts

---

## Structured Logging

```python
# config.py or logging setup
import logging
import json
from datetime import datetime


class JsonFormatter(logging.Formatter):
    """JSON log formatter for structured logging in Azure Monitor."""

    def format(self, record):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
        }
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_entry)


def configure_logging(level: str = "INFO"):
    """Configure structured JSON logging."""
    handler = logging.StreamHandler()
    handler.setFormatter(JsonFormatter())
    logging.basicConfig(level=getattr(logging, level), handlers=[handler])
```

---

## Required Dependencies

```
# For Python projects, add to requirements.txt:
opentelemetry-api>=1.24.0
opentelemetry-sdk>=1.24.0
azure-monitor-opentelemetry-exporter>=1.0.0b26
```

---

## Docker Environment Variables

Every container in `docker-compose.yml` and in Azure Container Apps must have:

```yaml
environment:
  - APPLICATION_INSIGHTS_CONNECTION_STRING=${APPLICATION_INSIGHTS_CONNECTION_STRING:-}
```

The connection string is output from the `monitoring.bicep` module and bound via `azd env`.

---

## Custom Trace Attributes

For domain-specific tracing, add custom attributes to spans:

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process-request") as span:
    span.set_attribute("request.id", request_id)
    span.set_attribute("request.type", request_type)
    # ... do work ...
    span.set_attribute("response.status", "success")
```

---

## Health Check Endpoints

Every service should expose a health check that observability can monitor:

```python
@app.get("/health")
async def health():
    return {"status": "healthy", "service": "<service-name>", "version": "1.0.0"}
```
