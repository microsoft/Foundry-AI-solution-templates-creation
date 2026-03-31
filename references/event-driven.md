# Event-Driven Microservices — Scaffold Pattern

This reference file defines the scaffold pattern for **message-based distributed systems** using Azure Service Bus, Event Grid, and Container Apps.

---

## Type-Specific Questions

| # | Question | Guidance |
|---|---|---|
| E1 | **Message broker?** | `Azure Service Bus` (default, queues + topics), `Azure Event Grid` (event routing), `Both` (hybrid). |
| E2 | **Service names and responsibilities?** | List each microservice: name, what events it produces, what events it consumes. |
| E3 | **Event schemas?** | `CloudEvents` (default, standard format), `Custom JSON`, `Avro`. |
| E4 | **Message patterns?** | `Pub/Sub` (default), `Request/Reply`, `Saga/Orchestration`, `Event Sourcing`. |
| E5 | **Per-service database?** | `Yes` (default, database-per-service), `Shared database`, `None` (stateless services). |
| E6 | **Dead-letter handling?** | `Auto-forward to DLQ` (default), `Custom DLQ processor`, `Alert and manual review`. |
| E7 | **KEDA scaling?** | `Yes` (default, scale on queue depth), `No` (fixed replicas). |
| E8 | **Idempotency strategy?** | `Idempotency key in message` (default), `Deduplication window`, `Database check`. |

---

## Project Folder Structure

```
<project-slug>/
├── services/
│   ├── <service-1>/
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── app/
│   │   │   ├── __init__.py
│   │   │   ├── main.py             # Service entry point + message consumer
│   │   │   ├── config.py           # Service configuration
│   │   │   ├── handlers/
│   │   │   │   ├── __init__.py
│   │   │   │   └── <event>.py      # Handler per event type consumed
│   │   │   ├── publishers/
│   │   │   │   ├── __init__.py
│   │   │   │   └── <event>.py      # Publisher per event type produced
│   │   │   ├── models/
│   │   │   │   ├── __init__.py
│   │   │   │   └── schemas.py      # Service-specific data models
│   │   │   └── db/
│   │   │       ├── __init__.py
│   │   │       └── client.py       # Per-service database client (if E5=Yes)
│   │   └── tests/
│   │       ├── __init__.py
│   │       └── test_handlers.py
│   │
│   └── <service-N>/               # (repeat for each service from E2)
│
├── shared/                         # Shared across all services
│   ├── __init__.py
│   ├── events/
│   │   ├── __init__.py
│   │   ├── base.py                 # Base event class (CloudEvents or custom)
│   │   └── catalog.py              # All event type definitions
│   ├── messaging/
│   │   ├── __init__.py
│   │   ├── publisher.py            # Abstract message publisher
│   │   ├── consumer.py             # Abstract message consumer
│   │   ├── servicebus.py           # Azure Service Bus implementation
│   │   └── eventgrid.py            # Azure Event Grid implementation (if E1 includes Event Grid)
│   └── idempotency/
│       ├── __init__.py
│       └── checker.py              # Idempotency check (from E8)
│
├── dlq-processor/                  # Dead-letter queue processor (if E6=Custom DLQ processor)
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── __init__.py
│       └── main.py
│
└── tests/
    ├── __init__.py
    └── integration/
        └── test_flow.py            # End-to-end message flow tests
```

---

## Source File Patterns

### Event Definitions

```python
# shared/events/base.py
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import uuid4

class CloudEvent(BaseModel):
    """CloudEvents v1.0 compliant event envelope."""
    specversion: str = "1.0"
    id: str = Field(default_factory=lambda: str(uuid4()))
    source: str
    type: str
    time: datetime = Field(default_factory=datetime.utcnow)
    datacontenttype: str = "application/json"
    data: dict

# shared/events/catalog.py
# Define all event types in the system
class EventTypes:
    ORDER_CREATED = "com.<project>.order.created"
    ORDER_PROCESSED = "com.<project>.order.processed"
    PAYMENT_COMPLETED = "com.<project>.payment.completed"
    NOTIFICATION_SENT = "com.<project>.notification.sent"
    # Add event types based on E2 answers
```

### Message Publisher (Service Bus)

```python
# shared/messaging/servicebus.py
import os
import json
from azure.servicebus.aio import ServiceBusClient
from azure.identity.aio import DefaultAzureCredential

class ServiceBusPublisher:
    """Publish messages to Azure Service Bus."""

    def __init__(self, queue_or_topic: str):
        self.namespace = os.environ["SERVICEBUS_NAMESPACE"]
        self.queue_or_topic = queue_or_topic
        self._credential = DefaultAzureCredential()

    async def publish(self, event: dict):
        """Publish a single event."""
        async with ServiceBusClient(
            f"{self.namespace}.servicebus.windows.net",
            credential=self._credential,
        ) as client:
            sender = client.get_topic_sender(self.queue_or_topic)
            async with sender:
                from azure.servicebus import ServiceBusMessage
                message = ServiceBusMessage(
                    body=json.dumps(event),
                    content_type="application/json",
                    message_id=event.get("id", ""),
                )
                await sender.send_messages(message)

class ServiceBusConsumer:
    """Consume messages from Azure Service Bus."""

    def __init__(self, queue_name: str, subscription_name: str | None = None):
        self.namespace = os.environ["SERVICEBUS_NAMESPACE"]
        self.queue_name = queue_name
        self.subscription_name = subscription_name
        self._credential = DefaultAzureCredential()

    async def consume(self, handler, max_messages: int = 10):
        """Consume and process messages."""
        async with ServiceBusClient(
            f"{self.namespace}.servicebus.windows.net",
            credential=self._credential,
        ) as client:
            if self.subscription_name:
                receiver = client.get_subscription_receiver(
                    self.queue_name, self.subscription_name,
                )
            else:
                receiver = client.get_queue_receiver(self.queue_name)

            async with receiver:
                messages = await receiver.receive_messages(
                    max_message_count=max_messages,
                    max_wait_time=30,
                )
                for msg in messages:
                    try:
                        event = json.loads(str(msg))
                        await handler(event)
                        await receiver.complete_message(msg)
                    except Exception as e:
                        # Message will be retried or sent to DLQ
                        await receiver.abandon_message(msg)
                        raise
```

### Service Entry Point

```python
# services/<service-name>/app/main.py
import asyncio
import logging
from .config import get_settings
from .handlers.<event> import handle_event
from shared.messaging.servicebus import ServiceBusConsumer

logger = logging.getLogger(__name__)

async def main():
    settings = get_settings()

    consumer = ServiceBusConsumer(
        queue_name=settings.input_queue,
        subscription_name=settings.subscription_name,
    )

    logger.info(f"Starting {settings.service_name}, listening on {settings.input_queue}")

    while True:
        try:
            await consumer.consume(handle_event)
        except Exception as e:
            logger.error(f"Error processing messages: {e}")
            await asyncio.sleep(5)  # Back off on error

if __name__ == "__main__":
    asyncio.run(main())
```

### Idempotency

```python
# shared/idempotency/checker.py
from collections import defaultdict
from datetime import datetime, timedelta

class IdempotencyChecker:
    """In-memory idempotency checker. Replace with database for production."""

    def __init__(self, window: timedelta = timedelta(hours=1)):
        self._processed: dict[str, datetime] = {}
        self._window = window

    def is_duplicate(self, message_id: str) -> bool:
        """Check if a message has already been processed."""
        self._cleanup()
        if message_id in self._processed:
            return True
        self._processed[message_id] = datetime.utcnow()
        return False

    def _cleanup(self):
        """Remove expired entries."""
        cutoff = datetime.utcnow() - self._window
        self._processed = {
            k: v for k, v in self._processed.items() if v > cutoff
        }
```

---

## KEDA Scaling (if E7=Yes)

Add KEDA scale rules to `container-app.bicep`:

```bicep
scale: {
  minReplicas: 0
  maxReplicas: 10
  rules: [
    {
      name: 'servicebus-rule'
      custom: {
        type: 'azure-servicebus'
        metadata: {
          queueName: '<queue-name>'
          namespace: '<servicebus-namespace>'
          messageCount: '5'  // Scale up when queue depth > 5
        }
        auth: [
          {
            secretRef: 'servicebus-connection'
            triggerParameter: 'connection'
          }
        ]
      }
    }
  ]
}
```

---

## Bicep Modules Required

- `monitoring.bicep` (always)
- `servicebus.bicep` — if E1 includes Service Bus
- `container-apps-env.bicep` + `container-app.bicep` (per service)
- `container-registry.bicep` (always)
- `cosmos.bicep` — if E5 = Yes and using Cosmos DB
- `keyvault.bicep` — for Service Bus connection strings and secrets

If E1 includes Event Grid:
- Event Grid topic/subscription resources in Bicep

---

## Type-Specific Quality Checklist

- [ ] All event types from E2 are defined in `shared/events/catalog.py`
- [ ] Each service only consumes events it's responsible for
- [ ] Each service publishes events with correct CloudEvents envelope (if E3=CloudEvents)
- [ ] Message handlers are idempotent (using strategy from E8)
- [ ] Dead-letter handling matches E6 specification
- [ ] KEDA scaling rules match E7 configuration (if applicable)
- [ ] Service-to-service communication is strictly through messages (no direct HTTP calls)
- [ ] Per-service databases are independent (no shared tables between services)
- [ ] Consumer processes handle poison messages without crashing
- [ ] Integration tests verify end-to-end message flow across services
- [ ] All Service Bus clients use `DefaultAzureCredential` (no connection strings)
