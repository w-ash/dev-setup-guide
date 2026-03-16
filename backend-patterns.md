# Backend Patterns: FastAPI, Repositories & Database

> **Scope**: Clean Architecture patterns for FastAPI — use case runner, thin route handlers, error envelopes, OpenAPI spec, batch-first repositories, Unit of Work lifecycle
> **Prerequisites**: [Python Tooling](python-tooling.md), [Project Structure](project-structure.md)
> **Deliverables**: `execute_use_case()` runner, route handlers delegating to use cases, error envelope middleware, repository protocols, Unit of Work
> **Estimated effort**: M

---

## Clean Architecture

```
Interface  →  Application  →  Domain  ←  Infrastructure
(FastAPI)     (Use Cases)     (Logic)    (DB, APIs)
```

---

## Use Case Runner Pattern

```python
# src/application/runner.py
from collections.abc import Callable, Coroutine
from typing import Any

from src.domain.repositories.interfaces import UnitOfWorkProtocol


async def execute_use_case[TResult](
    use_case_factory: Callable[[UnitOfWorkProtocol], Coroutine[Any, Any, TResult]],
) -> TResult:
    """Run a use case with proper session and UoW lifecycle.

    Lazy imports keep infrastructure out of the application layer's
    module-level namespace.
    """
    from src.infrastructure.persistence.database.db_connection import get_session
    from src.infrastructure.persistence.repositories.factories import get_unit_of_work

    async with get_session() as session:
        uow = get_unit_of_work(session)
        return await use_case_factory(uow)
```

Both CLI and API call the same runner — zero business logic duplication. See [CLI Patterns](cli-patterns.md) for how the CLI uses this same pattern. For what goes inside the use case (transaction ownership, repository access, testing), see [Use Case Architecture](use-case-architecture.md).

---

## Thin Route Handlers

```python
# src/interface/api/routes/items.py
from fastapi import APIRouter

from src.application.runner import execute_use_case
from src.application.use_cases.get_item import GetItemCommand, GetItemUseCase
from src.interface.api.schemas.items import ItemResponse

router = APIRouter(prefix="/items", tags=["items"])


@router.get("/{item_id}")
async def get_item(item_id: int) -> ItemResponse:
    result = await execute_use_case(
        lambda uow: GetItemUseCase(uow).execute(GetItemCommand(id=item_id))
    )
    return ItemResponse.from_domain(result)
```

Route handlers should be 5-10 lines. All business logic lives in use cases.

### Response Schema with `from_domain()`

```python
# src/interface/api/schemas/items.py
from pydantic import BaseModel

class ItemResponse(BaseModel):
    id: int
    name: str

    @classmethod
    def from_domain(cls, item: Item) -> "ItemResponse":
        return cls(id=item.id, name=item.name)
```

---

## Error Envelope

Consistent error responses across the entire API:

```python
# src/interface/api/middleware.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from src.domain.exceptions import NotFoundError, ValidationError


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(NotFoundError)
    async def not_found(_: Request, exc: NotFoundError) -> JSONResponse:
        return JSONResponse(
            status_code=404,
            content={"error": {"code": "NOT_FOUND", "message": str(exc)}},
        )

    @app.exception_handler(ValidationError)
    async def validation_error(_: Request, exc: ValidationError) -> JSONResponse:
        return JSONResponse(
            status_code=422,
            content={"error": {"code": "VALIDATION_ERROR", "message": str(exc)}},
        )

    @app.exception_handler(Exception)
    async def internal_error(_: Request, exc: Exception) -> JSONResponse:
        return JSONResponse(
            status_code=500,
            content={
                "error": {
                    "code": "INTERNAL_ERROR",
                    "message": "An internal error occurred",
                }
            },
        )
```

**Error shape**: `{"error": {"code": "UPPER_SNAKE", "message": "Human-readable description"}}`.
For paginated lists: `{"data": [...], "total": int, "limit": int, "offset": int}`.

---

## Application Factory

```python
# src/interface/api/app.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from src.interface.api.middleware import register_exception_handlers
from src.interface.api.routes.health import router as health_router
from src.interface.api.routes.items import router as items_router


def create_app() -> FastAPI:
    app = FastAPI(
        title="My Project",
        docs_url="/api/docs",
        openapi_url="/api/openapi.json",
    )

    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:5173"],  # Match your Vite port
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    register_exception_handlers(app)

    app.include_router(health_router, prefix="/api/v1")
    app.include_router(items_router, prefix="/api/v1")

    return app


app = create_app()
```

**FastAPI 0.135+ notes**:
- Strict Content-Type checking is now the default for JSON requests. Disable with `FastAPI(strict_content_type=False)` if clients send requests without proper headers.
- `ORJSONResponse` and `UJSONResponse` are deprecated — Pydantic's built-in JSON serialization is now faster (Rust-powered).
- Native SSE support via `StreamingResponse` — see [External API Resilience](external-api-resilience.md) for the full SSE pattern.

---

## OpenAPI Spec for Frontend Codegen

FastAPI auto-generates an OpenAPI spec at `/api/openapi.json`. Copy this to `web/openapi.json` for Orval codegen. Use `tags` on routers to control how Orval splits the generated code into separate files.

See [React API & Testing](react-api-testing.md) for the Orval configuration that consumes this spec.

---

**Skip the sections below** if your project doesn't use a database. [Domain Modeling](domain-modeling.md) and [External API Resilience](external-api-resilience.md) still apply.

---

## Repository Protocol Interfaces

Define repository contracts in the **domain layer** using `Protocol`. The key principle: **design for collections, not individual items**. Single-item operations are the degenerate case.

```python
# src/domain/repositories/interfaces.py
from typing import Protocol, Awaitable


class ItemRepositoryProtocol(Protocol):
    """Batch-first repository contract — collections over single items."""

    def find_by_ids(self, ids: list[int]) -> Awaitable[dict[int, Item]]:
        """Return dict keyed by ID for O(1) lookups."""
        ...

    def save_batch(self, items: list[Item]) -> Awaitable[list[Item]]:
        """Persist multiple items in one transaction."""
        ...

    def get_metrics_batch(
        self,
        ids: list[int],
        metric_type: str = "score",
    ) -> Awaitable[dict[int, float]]:
        """Aggregated metrics for many items in 1 query."""
        ...


class UnitOfWorkProtocol(Protocol):
    """Transaction boundary with repository access."""

    async def __aenter__(self) -> Self: ...
    async def __aexit__(self, *args: object) -> None: ...
    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...

    def get_item_repository(self) -> ItemRepositoryProtocol: ...
```

**Why dict returns**: `find_by_ids([1, 2, 3])` returns `{1: Item(...), 3: Item(...)}` — the caller instantly knows item 2 wasn't found.

---

## Batch-First Repository Implementation

```python
# src/infrastructure/persistence/repositories/items.py
from sqlalchemy import select
from sqlalchemy.orm import selectinload


class ItemRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def find_by_ids(self, ids: list[int]) -> dict[int, Item]:
        if not ids:
            return {}

        stmt = (
            select(DBItem)
            .where(DBItem.id.in_(ids))
            .options(selectinload(DBItem.tags))
        )
        result = await self._session.execute(stmt)
        rows = result.scalars().all()
        return {row.id: row.to_domain() for row in rows}
```

**Three critical patterns**:

1. **Empty-collection guards**: Always check `if not ids: return {}` before querying. Without this, `WHERE id IN ()` is invalid SQL in some engines.

2. **`selectinload()` for all relationships**: SQLAlchemy's default lazy loading fires a separate query per-item per-relationship. With 100 items x 2 relationships, that's 201 queries instead of 3. Use `joinedload()` only for single-object (many-to-one) relationships.

3. **Dict returns keyed by ID**: Callers get `O(1)` lookups and immediately see which requested IDs are missing.

### Bulk Insert with Duplicate Handling

```python
async def bulk_insert(self, items: list[Item]) -> tuple[int, int]:
    """Returns (inserted_count, duplicate_count)."""
    if not items:
        return (0, 0)

    inserted = 0
    duplicates = 0
    for item in items:
        try:
            self._session.add(DBItem.from_domain(item))
            await self._session.flush()
            inserted += 1
        except IntegrityError:
            await self._session.rollback()
            duplicates += 1

    return (inserted, duplicates)
```

---

## Unit of Work Lifecycle

The Unit of Work manages the transaction boundary: **auto-commit on success, auto-rollback on exception**.

```python
# src/infrastructure/persistence/unit_of_work.py
from typing import Self

class DatabaseUnitOfWork:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session
        self._committed = False

    async def __aenter__(self) -> Self:
        return self

    async def __aexit__(self, exc_type: type | None, *args: object) -> None:
        if exc_type is not None:
            await self.rollback()
        elif not self._committed:
            await self.commit()

    async def commit(self) -> None:
        await self._session.commit()
        self._committed = True

    async def rollback(self) -> None:
        await self._session.rollback()

    def get_item_repository(self) -> ItemRepository:
        return ItemRepository(self._session)
```

**Why auto-commit**: Forgetting `await uow.commit()` is a common bug that silently loses data. Auto-commit on clean exit makes "do work, return result" the happy path. Explicit `commit()` is still available for mid-transaction checkpoints.

For how use cases consume UoW (transaction ownership, commit/rollback rules, read-only patterns), see [Use Case Architecture](use-case-architecture.md).

---

## Logging Database Operations

```python
import functools
from loguru import logger


def db_operation(operation_name: str):
    """Decorator: log database operations with timing and error context."""

    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            logger.debug(f"DB {operation_name} started")
            try:
                result = await func(*args, **kwargs)
                logger.debug(f"DB {operation_name} completed")
                return result
            except Exception:
                logger.exception(f"DB {operation_name} failed")
                raise

        return wrapper

    return decorator
```

---

## Common Mistakes

**Mistake: Lazy-loading in async context**
```python
# x Triggers implicit SQL — fails in async SQLAlchemy
items = await repo.get_all()
for item in items:
    print(item.tags)  # Lazy load fires sync query -> exception

# ok Eager load in the repository query
stmt = select(DBItem).options(selectinload(DBItem.tags))
```

For async code, also consider SQLAlchemy's `AsyncAttrs` mixin — it provides awaitable attribute access (`await item.awaitable_attrs.tags`) as a cleaner alternative when eager loading isn't practical.

**Mistake: Single-item APIs that invite N+1 usage**
```python
# x Caller will loop and create N queries
async def get_by_id(self, id: int) -> Item | None: ...


# ok Batch API — caller sends all IDs at once
async def find_by_ids(self, ids: list[int]) -> dict[int, Item]: ...
```

**Mistake: No empty-collection guard**
```python
# x "WHERE id IN ()" -> SQL error or empty result with round-trip overhead
await session.execute(select(DBItem).where(DBItem.id.in_(ids)))

# ok Short-circuit before touching the database
if not ids:
    return {}
```
