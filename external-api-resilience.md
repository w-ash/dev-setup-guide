# External API Resilience

> **Scope**: Error classification, retry policies, HTTP client factories, SSE progress streaming
> **Prerequisites**: [Python Tooling](python-tooling.md), [Domain Modeling](domain-modeling.md)
> **Deliverables**: Error classifier hierarchy, retry policy factory, HTTP client with event hooks, SSE progress pattern
> **Estimated effort**: M

Resilience patterns for projects that call external APIs — with or without a database. Covers error classification, structured retry policies, HTTP client observability, and real-time progress streaming.

---

## Error Classification Protocol

Not all errors are equal. Classify them so retry logic can make intelligent decisions:

```python
# src/infrastructure/connectors/_shared/error_classifier.py
from typing import Protocol

class ErrorClassifier(Protocol):
    def classify(self, exception: Exception) -> tuple[str, str, str]:
        """Classify an error for retry behavior.

        Returns:
            (error_type, error_code, description)
            error_type: "permanent" | "temporary" | "rate_limit" | "not_found"
        """
        ...
```

### Base HTTP Classifier

Most external APIs communicate via HTTP. The base classifier maps status codes to error types:

```python
from abc import ABC, abstractmethod
import httpx

class HTTPErrorClassifier(ABC):
    """Shared HTTP error classification — override for service-specific rules."""

    def classify(self, exception: Exception) -> tuple[str, str, str]:
        # 1. Service-specific rules first (subclass hook)
        if result := self._classify_service_error(exception):
            return result

        # 2. HTTP status code mapping
        if isinstance(exception, httpx.HTTPStatusError):
            status = exception.response.status_code
            match status:
                case 404:
                    return ("not_found", "404", "Not found")
                case 429:
                    return ("rate_limit", "429", "Rate limited")
                case s if 400 <= s < 500:
                    return ("permanent", str(s), "Client error")
                case s if 500 <= s < 600:
                    return ("temporary", str(s), "Server error")

        # 3. Network errors are always temporary
        if isinstance(exception, httpx.RequestError):
            return ("temporary", "network", str(exception))

        # 4. Unknown — let retry logic decide
        return ("unknown", "N/A", str(exception))

    @abstractmethod
    def _classify_service_error(
        self, exception: Exception,
    ) -> tuple[str, str, str] | None:
        """Service-specific classification hook. Return None to fall through."""
        ...
```

### Service-Specific Overrides

Each external service overrides only what differs from HTTP defaults:

```python
class StripeErrorClassifier(HTTPErrorClassifier):
    """Stripe treats 401 as temporary (token refresh already triggered)."""

    def _classify_service_error(
        self, exception: Exception,
    ) -> tuple[str, str, str] | None:
        if (isinstance(exception, httpx.HTTPStatusError)
            and exception.response.status_code == 401
            and "expired" in exception.response.text.lower()):
            return ("temporary", "401", "Token expired — refreshing")
        return None  # Fall through to HTTP base
```

**Why a classifier hierarchy**: The base handles 90% of HTTP error logic. Each service overrides only its quirks. Adding a new API integration means writing 5-10 lines of service-specific classification, not reimplementing error handling from scratch.

---

## Retry Policies Driven by Classification

The classifier feeds into Tenacity retry policies. Permanent errors fail fast; temporary errors retry with backoff.

```python
# src/infrastructure/connectors/_shared/retry_policies.py
from tenacity import (
    AsyncRetrying, retry_if_exception, stop_after_attempt,
    wait_exponential, RetryCallState,
)

def create_classifier_retry(classifier: ErrorClassifier):
    """Build a retry predicate from an error classifier."""
    def should_retry(exc: BaseException) -> bool:
        if not isinstance(exc, Exception):
            return False  # Never retry KeyboardInterrupt, SystemExit
        error_type, _, _ = classifier.classify(exc)
        return error_type not in ("permanent", "not_found")
    return retry_if_exception(should_retry)
```

### Retry Configuration as Data

```python
from attrs import define

@define(frozen=True)
class RetryConfig:
    """All retry tuning in one place — no magic numbers in business logic."""
    service_name: str
    classifier: ErrorClassifier
    max_attempts: int = 3
    backoff_multiplier: float = 1.0
    backoff_max: float = 30.0
    max_total_delay: float | None = None

class RetryPolicyFactory:
    @staticmethod
    def create(config: RetryConfig) -> AsyncRetrying:
        return AsyncRetrying(
            stop=stop_after_attempt(config.max_attempts),
            wait=wait_exponential(
                multiplier=config.backoff_multiplier,
                max=config.backoff_max,
            ),
            retry=create_classifier_retry(config.classifier),
            before_sleep=_log_retry(config),
            reraise=True,
        )

def _log_retry(config: RetryConfig):
    """Log each retry attempt with classified error context."""
    def handler(retry_state: RetryCallState) -> None:
        exc = retry_state.outcome.exception() if retry_state.outcome else None
        if not isinstance(exc, Exception):
            return
        error_type, code, desc = config.classifier.classify(exc)
        if error_type == "rate_limit":
            logger.warning(f"{config.service_name} rate limited — pausing",
                           wait=retry_state.idle_for)
        else:
            logger.warning(f"{config.service_name} retry {retry_state.attempt_number}",
                           error_type=error_type, code=code)
    return handler
```

**Why config as data**: All retry behavior is visible in one object. No hunting through decorator chains to find "what's the max retry count?" Settings files can override `max_attempts` per service without code changes.

---

## HTTP Client Factories with Event Hooks

Create httpx clients through factories that attach structured logging hooks. Every request and response is logged automatically — no per-call logging code needed.

```python
# src/infrastructure/connectors/_shared/http_client.py
import httpx
from loguru import logger

_http_logger = logger.bind(module="http_client")

async def _log_request(request: httpx.Request) -> None:
    _http_logger.debug("HTTP request", method=request.method, url=str(request.url))

async def _log_response(response: httpx.Response) -> None:
    await response.aread()  # Buffer body + populate elapsed time
    if response.status_code < 400:
        _http_logger.debug("HTTP response",
                           status=response.status_code,
                           elapsed_ms=response.elapsed.total_seconds() * 1000)
    else:
        _http_logger.warning("HTTP error",
                             status=response.status_code,
                             body=response.text[:500],
                             retry_after=response.headers.get("Retry-After"))

_EVENT_HOOKS: dict[str, list] = {
    "request": [_log_request],
    "response": [_log_response],
}
```

### Service-Specific Client Factories

```python
def make_stripe_client(api_key: str) -> httpx.AsyncClient:
    return httpx.AsyncClient(
        base_url="https://api.stripe.com/v1",
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
        event_hooks=_EVENT_HOOKS,  # Shared hooks — all traffic logged
    )

def make_weather_client() -> httpx.AsyncClient:
    return httpx.AsyncClient(
        base_url="https://api.weather.gov",
        timeout=httpx.Timeout(connect=5.0, read=15.0, write=5.0, pool=5.0),
        event_hooks=_EVENT_HOOKS,
    )
```

**Why shared event hooks**: Every HTTP call in your application produces structured log entries — method, URL, status, latency, error body — without any per-call logging code. When debugging "why is this API slow?", the logs already have the answer.

---

## SSE Progress Streaming

For long-running operations triggered via API, return an **operation ID immediately** and stream progress via Server-Sent Events.

### Pattern: Pre-Assigned Operation IDs

```python
# src/interface/api/services/operations.py
from uuid import uuid4
import asyncio

class OperationRegistry:
    """Maps operation_id → asyncio.Queue for event routing."""

    def __init__(self) -> None:
        self._queues: dict[str, asyncio.Queue] = {}

    async def register(self, operation_id: str) -> asyncio.Queue:
        queue: asyncio.Queue = asyncio.Queue()
        self._queues[operation_id] = queue
        return queue

    async def get_queue(self, operation_id: str) -> asyncio.Queue | None:
        return self._queues.get(operation_id)

SSE_SENTINEL = object()  # Signals the SSE generator to close

async def prepare_operation() -> tuple[str, asyncio.Queue]:
    """Generate ID + register queue BEFORE starting work."""
    operation_id = str(uuid4())
    queue = await registry.register(operation_id)
    return operation_id, queue
```

### Route Handler: Return Immediately

```python
@router.post("/imports/data")
async def start_import(body: ImportRequest) -> OperationStartedResponse:
    operation_id, queue = await prepare_operation()

    # Launch background task — operation_id is pre-bound
    asyncio.create_task(run_import(operation_id, body))

    return OperationStartedResponse(operation_id=operation_id)
```

### SSE Endpoint: Stream from Queue

```python
@router.get("/operations/{operation_id}/progress")
async def stream_progress(operation_id: str) -> EventSourceResponse:
    queue = await registry.get_queue(operation_id)
    if queue is None:
        raise HTTPException(404, "Unknown operation")

    async def event_generator():
        while True:
            event = await queue.get()
            if event is SSE_SENTINEL:
                break
            yield event

    return EventSourceResponse(event_generator())
```

### Background Task: Emit Progress Events

```python
async def run_import(operation_id: str, request: ImportRequest) -> None:
    queue = await registry.get_queue(operation_id)
    try:
        for i, batch in enumerate(batches):
            await process_batch(batch)
            await queue.put({
                "event": "progress",
                "data": {"current": i + 1, "total": len(batches)},
            })
        await queue.put({"event": "complete", "data": {"status": "success"}})
    except Exception as e:
        await queue.put({"event": "error", "data": {"message": str(e)}})
    finally:
        await queue.put(SSE_SENTINEL)  # Close SSE connection
```

**Why pre-assigned IDs**: The client knows the operation ID *before* any work starts. It can immediately navigate to a progress view, reconnect after page refresh, or poll for status. The background task doesn't generate the ID — it receives it.

**Why queue-based routing**: Each operation has its own queue. In a multi-user system, progress events from operation A never leak to operation B's SSE stream.
