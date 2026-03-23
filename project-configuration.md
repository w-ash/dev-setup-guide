# Project Configuration

> **Scope**: Typed settings with Pydantic, constants organized by domain, structured logging with structlog
> **Prerequisites**: [Python Tooling](python-tooling.md)
> **Deliverables**: Settings module with typed fields, constants module, logging configuration
> **Estimated effort**: S

---

## Typed Settings with Pydantic

Use `pydantic-settings` with **Annotated types** for semantic validation. Invalid config fails at startup instead of at runtime.

```python
# src/config/settings.py
from typing import Annotated, ClassVar
from pydantic import BaseModel, Field
from pydantic_settings import BaseSettings, SettingsConfigDict

# Semantic type aliases — self-documenting and reusable
Percentage = Annotated[float, Field(ge=0.0, le=1.0)]
PositiveFloat = Annotated[float, Field(gt=0.0)]
PositiveInt = Annotated[int, Field(gt=0)]
ConfidenceScore = Annotated[int, Field(ge=0, le=100)]
```

### Nested Configuration Groups

```python
class DatabaseConfig(BaseModel):
    url: str = "sqlite+aiosqlite:///data/app.db"
    echo: bool = False


class LoggingConfig(BaseModel):
    console_level: str = "INFO"
    file_level: str = "DEBUG"
    json_output: bool = True


class APIConfig(BaseModel):
    batch_size: PositiveInt = 50
    rate_limit: PositiveFloat | None = None
    request_timeout: PositiveInt = 30


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_nested_delimiter="__",  # DATABASE__URL → database.url
        case_sensitive=False,
        env_ignore_empty=True,  # FOO= uses default, not empty string
    )

    database: DatabaseConfig = DatabaseConfig()
    logging: LoggingConfig = LoggingConfig()
    api: APIConfig = APIConfig()
```

### Flat Environment Variable Routing

Support both nested (`DATABASE__URL`) and flat (`DATABASE_URL`) environment variables for backward compatibility with scripts and deployment tools:

```python
from pydantic import model_validator


class Settings(BaseSettings):
    # Map flat env vars to nested structure
    _FLAT_ROUTES: ClassVar[dict[str, tuple[str, str]]] = {
        "DATABASE_URL": ("database", "url"),
        "LOG_LEVEL": ("logging", "console_level"),
    }

    @model_validator(mode="before")
    @classmethod
    def route_flat_env_vars(cls, data: dict) -> dict:
        """Route DATABASE_URL → database.url for backward compat."""
        import os

        for env_var, (group, field) in cls._FLAT_ROUTES.items():
            if value := os.environ.get(env_var):
                data.setdefault(group, {})[field] = value
        return data
```

**Why Annotated types**: `batch_size: PositiveInt = 50` is self-documenting — the type constraint *is* the validation. No separate validator function needed. Type checkers see `int`, Pydantic enforces `> 0`.

**Why `env_ignore_empty`**: Without this, setting `FOO=` in your shell overrides the default with an empty string, which silently breaks string-to-int conversions.

---

## Constants Organized by Domain

Group constants by *concern*, not by type. Use classes as namespaces with `Final` typing.

```python
# src/config/constants.py
from typing import Final


class HTTPStatus:
    """HTTP status range boundaries for error classification."""

    CLIENT_ERROR_MIN: Final = 400
    CLIENT_ERROR_MAX: Final = 500
    SERVER_ERROR_MIN: Final = 500
    SERVER_ERROR_MAX: Final = 600


class BusinessLimits:
    """Business logic limits — not user-configurable."""

    DEFAULT_PAGE_SIZE: Final = 50
    MAX_PAGE_SIZE: Final = 200
    MAX_BATCH_SIZE: Final = 1000
    MAX_RETRY_ATTEMPTS: Final = 3


class ExternalAPI:
    """External API format specifications and validation constants."""

    MAX_SEARCH_RESULTS: Final = 10
    IDENTIFIER_LENGTH: Final = 22
    URI_PARTS: Final = 3


class Defaults:
    """Default values used across the application."""

    DEFAULT_USER_ID: Final = "default"
    DEFAULT_TIMEZONE: Final = "UTC"
```

**Why classes over module-level constants**:
- `BusinessLimits.MAX_PAGE_SIZE` is self-documenting — you know the domain
- IDE autocomplete: type `BusinessLimits.` and see all business limits
- Each class has a docstring explaining its scope
- `Final` prevents accidental reassignment

**Anti-pattern**: Scattering `MAX_PAGE_SIZE = 200` across 5 files. When you need to change it, which one is authoritative?

---

## Structured Logging with structlog

### Why structlog in 2026

structlog (v25.5+) is the recommended structured logging library for Python. Three reasons:

1. **Flat JSON by default** — `JSONRenderer()` produces `{"level": "info", "event": "msg"}`. AI agents, jq, and log aggregators (Loki, Datadog) parse top-level keys directly. No `.record.level.name` traversal.
2. **Stdlib integration mode** — structlog wraps Python's `logging` module, so Prefect, Uvicorn, FastAPI, and any third-party library using stdlib loggers flow through the same processor pipeline automatically. Zero bridge code.
3. **Composable processor pipeline** — one chain of functions handles timestamping, context merging, callsite info, and rendering. Swap `ConsoleRenderer` for `JSONRenderer` per handler — same processors, different output.

### Configuration

Configure logging once at startup. Use **stdlib integration mode** so all Python loggers (Prefect, Uvicorn, etc.) flow through structlog's processor pipeline automatically.

```python
# src/config/logging.py
import logging
import logging.handlers
import sys
from pathlib import Path

import structlog


def setup_logging(*, verbose: bool = False) -> None:
    """Configure structlog with dual output: pretty console + flat JSON file."""
    console_level = "DEBUG" if verbose else "INFO"

    # Shared processor chain — runs for ALL output (console + file + third-party)
    shared_processors = [
        structlog.contextvars.merge_contextvars,  # async-safe context propagation
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.ExtraAdder(),             # stdlib extra → structlog event dict
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.CallsiteParameterAdder([
            structlog.processors.CallsiteParameter.FUNC_NAME,
            structlog.processors.CallsiteParameter.LINENO,
        ]),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.UnicodeDecoder(),
    ]

    # Configure structlog to wrap stdlib logging
    structlog.configure(
        processors=[*shared_processors, structlog.stdlib.ProcessorFormatter.wrap_for_formatter],
        wrapper_class=structlog.stdlib.BoundLogger,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

    # Console handler — colorized for humans
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(getattr(logging, console_level))
    console_handler.setFormatter(structlog.stdlib.ProcessorFormatter(
        foreign_pre_chain=shared_processors,  # process third-party stdlib logs too
        processors=[
            structlog.stdlib.ProcessorFormatter.remove_processors_meta,
            structlog.dev.ConsoleRenderer(colors=True),
        ],
    ))

    # File handler — flat JSON for agents, jq, log aggregation
    file_handler = logging.handlers.RotatingFileHandler("logs/app.log", maxBytes=10_485_760, backupCount=7)
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(structlog.stdlib.ProcessorFormatter(
        foreign_pre_chain=shared_processors,
        processors=[
            structlog.stdlib.ProcessorFormatter.remove_processors_meta,
            structlog.processors.dict_tracebacks,  # structured exception dicts
            structlog.processors.JSONRenderer(),
        ],
    ))

    # Attach to root logger — ALL Python loggers flow through this
    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(console_handler)
    root.addHandler(file_handler)
    root.setLevel(logging.DEBUG)
```

The JSON output is flat — every field is a top-level key:

```json
{
  "timestamp": "2026-03-22T10:00:00.000000Z",
  "level": "info",
  "event": "Processing item",
  "logger": "app.services.importer",
  "func_name": "process_item",
  "lineno": 42,
  "service": "my-app",
  "item_id": 123
}
```

### Module-Level Logger Binding

```python
# In any module:
import structlog

logger = structlog.get_logger(__name__, service="my-app")


async def process_item(item_id: int) -> None:
    logger.info("Processing item", item_id=item_id)
    # Flat JSON: {"event": "Processing item", "item_id": 123, "logger": "app.services", ...}
```

Or use a factory that pre-binds common context:

```python
# src/config/logging.py
def get_logger(name: str) -> structlog.stdlib.BoundLogger:
    return structlog.get_logger(name, service="my-app", module=name)
```

### Context Binding for Operations

**Per-logger binding** (permanent for that logger instance):

```python
async def import_data(source: str, user_id: str) -> None:
    op_log = logger.bind(operation="import", source=source, user_id=user_id)
    op_log.info("Import started")

    for batch in batches:
        op_log.debug("Processing batch", batch_size=len(batch))

    op_log.info("Import completed", total=total_count)
```

**Async-safe context propagation** (via contextvars — propagates across `await` boundaries):

```python
from contextlib import contextmanager
from structlog.contextvars import bind_contextvars, unbind_contextvars

@contextmanager
def logging_context(**kwargs):
    bind_contextvars(**kwargs)
    try:
        yield
    finally:
        unbind_contextvars(*kwargs.keys())

# Usage — all logs inside this block carry workflow_id, even across await boundaries
with logging_context(workflow_id=42, run_id="abc123"):
    logger.info("Starting workflow")          # includes workflow_id, run_id
    await some_async_operation()              # logs inside also include them
    logging.getLogger("prefect").info("Task done")  # stdlib logs too!
```

### Exception Logging

structlog uses stdlib's `exc_info` parameter — no special API needed:

```python
try:
    await risky_operation()
except Exception:
    logger.error("Operation failed", exc_info=True, operation="sync")
    # dict_tracebacks processor renders structured exception dicts in JSON output
```
