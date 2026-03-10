# Python 3.14+ Coding Patterns

> **Scope**: Modern Python syntax patterns — 8 PEPs with DO/DON'T examples
> **Prerequisites**: [Python Tooling](python-tooling.md)
> **Deliverables**: Team familiarity with modern Python idioms; Ruff and BasedPyright configured to enforce them
> **Estimated effort**: S (reference material, not implementation)

Each pattern shows the modern way and the legacy way it replaces.

---

## Generics (PEP 695)
```python
# DO — Python 3.14 syntax
class Repository[TModel, TDomain]:
    ...

async def execute[TResult](factory: Callable[..., TResult]) -> TResult:
    ...

# DON'T — legacy
from typing import Generic, TypeVar
T = TypeVar("T")
class Repository(Generic[T]):
    ...
```

## Union Types (PEP 604)
```python
# DO
def find(id: int) -> User | None: ...
def parse(value: str | int | float) -> str: ...

# DON'T
from typing import Optional, Union
def find(id: int) -> Optional[User]: ...
```

## PEP 649 Deferred Annotations
```python
# DO — Python 3.14 evaluates annotations lazily by default
def process(item: MyClass) -> MyClass:
    ...

# DON'T
from __future__ import annotations     # Unnecessary in 3.14
def process(item: "MyClass") -> "MyClass":  # String quotes unnecessary
    ...
```

## Timestamps (PEP 615)
```python
# DO
from datetime import UTC, datetime
now = datetime.now(UTC)

# DON'T
now = datetime.utcnow()     # Returns naive datetime (deprecated)
now = datetime.now()         # Returns local time (ambiguous)
```

## Structured Concurrency (PEP 654)
```python
# DO — TaskGroup cancels all on first failure
async with asyncio.TaskGroup() as tg:
    tg.create_task(fetch_users())
    tg.create_task(fetch_orders())

# DON'T — gather leaves partial results on failure
results = await asyncio.gather(fetch_users(), fetch_orders())
```

## Multi-Exception Handling (PEP 758)
```python
# DO — Python 3.14, no parentheses needed
except TimeoutError, ConnectionError:
    handle_network_error()

# Previous style (still works)
except (TimeoutError, ConnectionError):
    handle_network_error()
```

## Type Guards (PEP 742)
```python
# DO
from typing import TypeIs

def is_admin(user: User) -> TypeIs[AdminUser]:
    return user.role == "admin"

if is_admin(user):
    user.admin_action()  # Type narrowed to AdminUser

# DON'T
if hasattr(user, "admin_action"):  # type: ignore
    user.admin_action()
```

## Structured Logging (loguru)
```python
# DO
from loguru import logger
log = logger.bind(service="auth", user_id=user.id)
log.info("Login successful")
log.opt(exception=True).error("Authentication failed")

# DON'T
import logging
logger = logging.getLogger(__name__)
logger.error("Failed", exc_info=True)  # loguru ignores exc_info kwarg
```
