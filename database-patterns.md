# Database Patterns

> **Scope**: Batch-first repositories, Unit of Work lifecycle, eager loading, repository protocols
> **Prerequisites**: [FastAPI Backend](fastapi-backend.md), [Domain Modeling](domain-modeling.md)
> **Deliverables**: Repository protocol interfaces, batch-first implementations, Unit of Work with auto-commit
> **Estimated effort**: M

**Skip this file** if your project doesn't use a database (API gateways, CLI tools wrapping external services, serverless functions). [Domain Modeling](domain-modeling.md) and [External API Resilience](external-api-resilience.md) still apply.

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
        self, ids: list[int], metric_type: str = "score",
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

**Why dict returns**: `find_by_ids([1, 2, 3])` returns `{1: Item(...), 3: Item(...)}` — the caller instantly knows item 2 wasn't found. Compare to returning a list where you'd need `O(n)` scanning to check for missing items.

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
        # Empty-collection guard — avoids sending "WHERE id IN ()" to the DB
        if not ids:
            return {}

        stmt = (
            select(DBItem)
            .where(DBItem.id.in_(ids))
            .options(selectinload(DBItem.tags))  # Eager load relationships
        )
        result = await self._session.execute(stmt)
        rows = result.scalars().all()
        return {row.id: row.to_domain() for row in rows}
```

**Three critical patterns**:

1. **Empty-collection guards**: Always check `if not ids: return {}` before querying. Without this, `WHERE id IN ()` is invalid SQL in some engines and wasteful in all of them.

2. **`selectinload()` for all relationships**: SQLAlchemy's default lazy loading fires a separate query per-item per-relationship. With 100 items × 2 relationships, that's 201 queries instead of 3. Use `joinedload()` only for single-object (many-to-one) relationships — it uses JOINs which cause cartesian products with collections.

3. **Dict returns keyed by ID**: Callers get `O(1)` lookups and can immediately see which requested IDs are missing.

### Bulk Insert with Duplicate Handling

```python
async def bulk_insert(self, items: list[Item]) -> tuple[int, int]:
    """Returns (inserted_count, duplicate_count) — not just success/fail."""
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
            await self.rollback()           # Exception → rollback
        elif not self._committed:
            await self.commit()             # Clean exit → auto-commit

    async def commit(self) -> None:
        await self._session.commit()
        self._committed = True

    async def rollback(self) -> None:
        await self._session.rollback()

    def get_item_repository(self) -> ItemRepository:
        return ItemRepository(self._session)
```

**Why auto-commit**: Forgetting `await uow.commit()` is a common bug that silently loses data. Auto-commit on clean exit makes "do work, return result" the happy path. Explicit `commit()` is still available for use cases that need mid-transaction checkpoints.

### The `execute_use_case()` Bridge

A single function connects the UoW lifecycle to use case execution:

```python
# src/application/runner.py
async def execute_use_case[TResult](
    use_case_factory: Callable[[UnitOfWorkProtocol], Coroutine[Any, Any, TResult]],
) -> TResult:
    """Run a use case with proper session and UoW lifecycle."""
    from src.infrastructure.persistence.database.db_connection import get_session
    from src.infrastructure.persistence.repositories.factories import get_unit_of_work

    async with get_session() as session:
        uow = get_unit_of_work(session)
        return await use_case_factory(uow)

# Usage in route handler:
result = await execute_use_case(
    lambda uow: CreateOrderUseCase().execute(CreateOrderCommand(...), uow)
)
```

Both CLI and API call `execute_use_case()` — zero business logic duplication between interfaces.

---

## Logging Database Operations

Wrap repository methods with structured logging for diagnostics:

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

# Usage
class ItemRepository:
    @db_operation("find_by_ids")
    async def find_by_ids(self, ids: list[int]) -> dict[int, Item]:
        ...
```

---

## Common Mistakes

**Mistake: Lazy-loading in async context**
```python
# ❌ Triggers implicit SQL — fails in async SQLAlchemy
items = await repo.get_all()
for item in items:
    print(item.tags)  # Lazy load fires sync query → exception

# ✓ Eager load in the repository query
stmt = select(DBItem).options(selectinload(DBItem.tags))
```

**Mistake: Single-item APIs that invite N+1 usage**
```python
# ❌ Caller will loop and create N queries
async def get_by_id(self, id: int) -> Item | None: ...

# ✓ Batch API — caller sends all IDs at once
async def find_by_ids(self, ids: list[int]) -> dict[int, Item]: ...
```

**Mistake: No empty-collection guard**
```python
# ❌ "WHERE id IN ()" → SQL error or empty result with round-trip overhead
await session.execute(select(DBItem).where(DBItem.id.in_(ids)))

# ✓ Short-circuit before touching the database
if not ids:
    return {}
```
