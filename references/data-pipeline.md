# Data Pipeline — Scaffold Pattern

This reference file defines the scaffold pattern for **ETL/ELT data processing pipelines** deployed on Azure.

---

## Type-Specific Questions

| # | Question | Guidance |
|---|---|---|
| D1 | **Processing mode?** | `batch` (default), `streaming`, `hybrid`. Drives pipeline architecture. |
| D2 | **Data sources?** | List each source: type (database, API, file system, event stream), format (CSV, JSON, Parquet, Avro), location. |
| D3 | **Data sinks/destinations?** | List each sink: Azure Data Lake, Cosmos DB, SQL Database, Blob Storage, Synapse. |
| D4 | **Transformation engine?** | `Python/Pandas` (default for small data), `PySpark` (large data), `Azure Data Factory` (low-code), `dbt` (SQL transforms). |
| D5 | **Scheduling?** | `Timer/cron` (default), `Event-triggered`, `Manual`, `Azure Data Factory triggers`. |
| D6 | **Data volume per run?** | Small (<1GB), Medium (1-100GB), Large (>100GB). Drives compute sizing and parallelism. |
| D7 | **Data quality checks?** | `Basic` (null checks, type validation), `Advanced` (schema evolution, anomaly detection), `None`. |
| D8 | **Orchestration tool?** | `Python scripts` (default), `Azure Data Factory`, `Apache Airflow`, `Prefect`. |

---

## Project Folder Structure

```
<project-slug>/
├── src/
│   ├── __init__.py
│   ├── config.py                   # Pipeline configuration
│   ├── pipeline.py                 # Main pipeline orchestrator
│   ├── extractors/
│   │   ├── __init__.py
│   │   ├── base.py                 # Abstract extractor interface
│   │   └── <source>.py             # One extractor per data source (from D2)
│   ├── transformers/
│   │   ├── __init__.py
│   │   ├── base.py                 # Abstract transformer interface
│   │   ├── cleaners.py             # Data cleaning transformations
│   │   ├── enrichers.py            # Data enrichment transformations
│   │   └── validators.py           # Data quality checks (from D7)
│   ├── loaders/
│   │   ├── __init__.py
│   │   ├── base.py                 # Abstract loader interface
│   │   └── <sink>.py               # One loader per destination (from D3)
│   └── utils/
│       ├── __init__.py
│       ├── logging.py              # Structured logging
│       └── metrics.py              # Pipeline metrics collection
│
├── pipelines/                      # Pipeline definitions
│   ├── <pipeline-name>.yaml        # Pipeline config: sources, transforms, sinks
│   └── schedules.yaml              # Scheduling configuration (from D5)
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_extractors.py
│   ├── test_transformers.py
│   └── test_loaders.py
│
└── data/
    ├── sample/                     # Sample input data for testing
    └── schemas/                    # JSON Schema or Avro schema definitions
```

---

## Source File Patterns

### Pipeline Orchestrator

```python
# src/pipeline.py
import logging
from dataclasses import dataclass
from .config import PipelineConfig
from .extractors.base import Extractor
from .transformers.base import Transformer
from .loaders.base import Loader

logger = logging.getLogger(__name__)

@dataclass
class PipelineResult:
    records_extracted: int
    records_transformed: int
    records_loaded: int
    errors: list[str]

async def run_pipeline(config: PipelineConfig) -> PipelineResult:
    """Execute the ETL pipeline: extract → transform → load."""
    errors = []

    # Extract
    logger.info(f"Extracting from {config.source_name}")
    extractor = Extractor.create(config.source_type, config.source_config)
    raw_data = await extractor.extract()
    logger.info(f"Extracted {len(raw_data)} records")

    # Transform
    logger.info("Running transformations")
    transformer = Transformer.create(config.transform_steps)
    transformed_data, transform_errors = await transformer.transform(raw_data)
    errors.extend(transform_errors)
    logger.info(f"Transformed {len(transformed_data)} records ({len(transform_errors)} errors)")

    # Validate (if D7 != None)
    if config.validation_enabled:
        from .transformers.validators import validate
        validated_data, validation_errors = validate(transformed_data, config.schema)
        errors.extend(validation_errors)
        transformed_data = validated_data

    # Load
    logger.info(f"Loading to {config.sink_name}")
    loader = Loader.create(config.sink_type, config.sink_config)
    loaded_count = await loader.load(transformed_data)
    logger.info(f"Loaded {loaded_count} records")

    return PipelineResult(
        records_extracted=len(raw_data),
        records_transformed=len(transformed_data),
        records_loaded=loaded_count,
        errors=errors,
    )
```

### Extractor Base

```python
# src/extractors/base.py
from abc import ABC, abstractmethod
from typing import Any

class Extractor(ABC):
    @abstractmethod
    async def extract(self) -> list[dict[str, Any]]:
        """Extract data from source. Returns list of records."""
        ...

    @staticmethod
    def create(source_type: str, config: dict) -> "Extractor":
        """Factory: create extractor by type."""
        extractors = {
            "blob_storage": "BlobStorageExtractor",
            "cosmos_db": "CosmosDBExtractor",
            "rest_api": "RestAPIExtractor",
            "csv_file": "CSVExtractor",
        }
        # Dynamic import and instantiation
        ...
```

---

## Bicep Modules Required

- `monitoring.bicep` (always)
- `storage.bicep` — for Data Lake / Blob Storage (almost always needed)
- `container-apps-env.bicep` + `container-app.bicep` — if running as containerized jobs
- `cosmos.bicep` — if D3 includes Cosmos DB

If D8 = Azure Data Factory:
- `data-factory.bicep` module (additional)

If D4 = PySpark:
- `databricks.bicep` or `synapse.bicep` module (additional)

---

## Type-Specific Quality Checklist

- [ ] All extractors handle connection failures with retries
- [ ] All loaders implement idempotent upserts (no duplicate data on re-run)
- [ ] Data quality validators match D7 answer
- [ ] Pipeline configuration is externalized (YAML or env vars, not hardcoded)
- [ ] Scheduling matches D5 answer
- [ ] Error handling captures and logs all failed records without stopping pipeline
- [ ] Sample data in `data/sample/` exercises all code paths
- [ ] Pipeline metrics are collected and sent to Application Insights
- [ ] Schema definitions exist for all data formats
- [ ] Tests mock external data sources and sinks
