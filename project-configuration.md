# Project Configuration

> **Scope**: Typed settings with Pydantic, constants organized by domain, structured logging with loguru
> **Prerequisites**: [Python Tooling](python-tooling.md)
> **Deliverables**: Settings module with typed fields, constants module, logging configuration
> **Estimated effort**: S

Foundation patterns every Python project benefits from: typed configuration that validates at startup, organized constants, and structured logging. No database required.

---

## Typed Settings with Pydantic

Use `pydantic-settings` with **Annotated types** for semantic validation. Invalid config fails at startup, not at 3 AM in production.

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

## Structured Logging with Loguru

Configure logging once at startup. Separate **console** (human-readable, colorized) from **file** (structured JSON, machine-parseable) output.

```python
# src/config/logging.py
import sys
from loguru import logger
from pathlib import Path


def setup_logging(*, verbose: bool = False, log_dir: str = "logs") -> None:
    """Configure logging with console + JSON file output."""
    log_path = Path(log_dir)
    log_path.mkdir(exist_ok=True)

    console_level = "DEBUG" if verbose else "INFO"

    logger.configure(
        handlers=[
            # Console: colorized, real-time debugging
            {
                "sink": sys.stdout,
                "level": console_level,
                "format": (
                    "<green>{time:HH:mm:ss.SSS}</green> | "
                    "<level>{level: <8}</level> | "
                    "<cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - "
                    "<level>{message}</level>"
                ),
                "colorize": True,
            },
            # File: structured JSON, rotated, compressed
            {
                "sink": str(log_path / "app.log"),
                "level": "DEBUG",
                "serialize": True,  # JSON output for log aggregation
                "rotation": "10 MB",  # Size-based rotation
                "retention": "1 week",  # Auto-delete old logs
                "compression": "zip",  # Compress rotated files
                "enqueue": True,  # Async writes (don't block app)
                "catch": True,  # Don't crash app on logging errors
            },
        ],
        # Default context fields — appear in every log entry
        extra={"service": "my-app", "module": "root"},
    )
```

### Module-Level Logger Binding

```python
# In any module:
from loguru import logger

log = logger.bind(module=__name__)


async def process_item(item_id: int) -> None:
    log.info("Processing item", item_id=item_id)
    # JSON output: {"message": "Processing item", "extra": {"module": "app.services", "item_id": 42}}
```

### Context Binding for Operations

```python
async def import_data(source: str, user_id: str) -> None:
    # Bind context once — all subsequent log calls include it
    op_log = logger.bind(operation="import", source=source, user_id=user_id)
    op_log.info("Import started")

    for batch in batches:
        op_log.debug("Processing batch", batch_size=len(batch))
        # Every log entry now carries operation, source, user_id

    op_log.info("Import completed", total=total_count)
```

**Why `enqueue=True`**: Without this, file writes block the event loop in async applications. `enqueue=True` sends log entries to a background thread, keeping your async code responsive.

**Why `serialize=True`**: JSON logs can be ingested by log aggregation tools (Grafana Loki, ELK, CloudWatch). Structured fields (`item_id`, `operation`) become searchable dimensions instead of buried in free-text messages.

### Security: Disable Backtrace in Production

```python
# Production handler — no backtrace or diagnose (prevents info leakage)
{
    "sink": str(log_path / "app.log"),
    "backtrace": False,  # Don't show local variables in tracebacks
    "diagnose": False,  # Don't show variable values in exception frames
}

# Development handler — full diagnostics
{
    "sink": sys.stdout,
    "backtrace": True,  # Show local variables
    "diagnose": True,  # Show variable values (sensitive data risk!)
}
```
