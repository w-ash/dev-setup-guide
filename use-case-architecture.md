# Use Case Architecture

> **Scope**: Use case file anatomy, transaction ownership, repository access, external service protocols, audit checklist
> **Prerequisites**: [Domain Modeling](domain-modeling.md)
> **Deliverables**: Use cases following the Command/Result/Execute contract, passing the audit checklist
> **Estimated effort**: M

How to structure use cases so they're testable, composable, and auditable. The runner and route handler patterns are in [Backend Patterns](backend-patterns.md) — this guide covers what happens *inside* the use case. For Command/Result data modeling patterns (attrs, validators, sentinel for partial updates), see [Domain Modeling](domain-modeling.md).

---

## The Contract

Every use case has exactly one entry point with exactly one signature:

```python
async def execute(self, command: CommandType, uow: UnitOfWorkProtocol) -> ResultType
```

This never varies. When you need new parameters, add fields to the Command — not new arguments to `execute()`. This means every caller (CLI, API, tests, other use cases) uses the same interface, and adding a parameter is backward-compatible by default.

---

## File Anatomy: Three Objects

Every use case file contains exactly three `@define` classes:

```python
# src/application/use_cases/create_order.py

@define(frozen=True, slots=True)
class CreateOrderCommand:
    """Immutable input — validated at construction, never modified."""

    customer_id: str = field(validator=non_empty_string)
    items: list[LineItem]
    priority: str = "normal"


@define(frozen=True, slots=True)
class CreateOrderResult:
    """Immutable output — snapshot of what happened."""

    order: Order
    items_processed: int
    warnings: list[str] = field(factory=list)


@define(slots=True)
class CreateOrderUseCase:
    """Mutable orchestrator — stateless between calls."""

    async def execute(
        self, command: CreateOrderCommand, uow: UnitOfWorkProtocol
    ) -> CreateOrderResult:
        async with uow:
            # ... business logic ...
            await uow.commit()
            return CreateOrderResult(...)
```

| Object | Decorator | Why |
|--------|-----------|-----|
| **Command** | `@define(frozen=True, slots=True)` | Crosses layer boundaries — must be immutable |
| **Result** | `@define(frozen=True, slots=True)` | Returned to callers — must be immutable |
| **UseCase** | `@define(slots=True)` | May hold injected services, but stateless between calls |

Even parameterless queries get an empty Command for API uniformity:

```python
@define(frozen=True, slots=True)
class ListOrdersCommand:
    """Parameterless — exists so the signature never changes when params are added."""
```

`frozen=True` on Command/Result prevents shared mutable state across layer boundaries. `slots=True` everywhere prevents silent typos like `command.cusotmer_id` from creating dynamic attributes.

---

## Transaction Ownership

The use case owns the transaction — not the runner, not the route handler:

```python
async def execute(self, command, uow):
    async with uow:                    # Opens transaction
        repo = uow.get_order_repository()
        order = await repo.create(command.items)
        await uow.commit()             # Explicit commit on success
        return CreateOrderResult(order=order, items_processed=len(command.items))
    # Exception escaping the block → auto-rollback, no try/catch needed
```

Rules:
- **`async with uow:` is mandatory** — scopes the transaction
- **`await uow.commit()` is explicit** — only on the success path
- **Read-only use cases skip `commit()`** — still use `async with uow:` for session scoping
- **Exceptions auto-rollback** — the context manager handles cleanup

For UoW implementation details (auto-commit, session lifecycle, `execute_use_case()` bridge), see [Backend Patterns](backend-patterns.md).

### Read-Only Pattern (Previews, Lists, Gets)

```python
async def execute(self, command, uow):
    async with uow:
        items = await uow.get_item_repository().list_all()
        return ListItemsResult(items=items, total_count=len(items))
        # No commit() — read-only
```

Preview use cases fetch external state and compute diffs but never call `commit()`. They answer "what would happen if...?" without side effects.

---

## Repository Access: Always Through UoW

Repositories are **never** injected into use case constructors. Always fetch them inside `execute()`:

```python
async with uow:
    order_repo = uow.get_order_repository()
    customer_repo = uow.get_customer_repository()
    inventory_repo = uow.get_inventory_repository()
```

Why: all repositories from the same UoW share the same database session. If you injected repos in the constructor, you'd have to manually coordinate sessions — a problem the UoW pattern solves for free.

---

## External Services: Typed Connector Protocols

When a use case calls external APIs, it uses **typed resolver functions** that return capability-specific protocols — not raw infrastructure imports:

```python
# Define capability protocols in application layer
class StorageConnector(Protocol):
    async def get_folder(self, folder_id: str) -> ExternalFolder: ...
    async def upload_items(self, folder_id: str, items: list) -> None: ...


# Resolver functions return typed capabilities
def resolve_storage_connector(name: str, uow: UnitOfWorkProtocol) -> StorageConnector:
    """Typed resolver — returns concrete connector cast to protocol."""
    return cast(StorageConnector, resolve_connector(name, uow))


# Use case consumes the protocol, not the implementation
class SyncToStorageUseCase:
    async def execute(self, command, uow):
        connector: StorageConnector = resolve_storage_connector(command.service, uow)
        folder = await connector.get_folder(command.external_id)
```

---

## Command Validation

Commands validate inputs at construction time using attrs validators:

```python
from src.application.use_cases._shared.command_validators import (
    non_empty_string,
    positive_int_in_range,
)


@define(frozen=True, slots=True)
class CreateOrderCommand:
    customer_id: str = field(validator=non_empty_string)
    quantity: int = field(validator=positive_int_in_range(1, 10000))
    notes: str = ""  # Optional fields don't need validators
```

Principle: **if a Command can be constructed, it's valid.** The use case body never checks "is this string empty?" — that was enforced at construction.

### Reusable Validators

Keep validators in a shared module (`_shared/command_validators.py`):

```python
def non_empty_string(instance, attribute, value):
    if not value or not value.strip():
        raise ValueError(f"{attribute.name} must be non-empty, got: {value!r}")


def positive_int_in_range(min_val=1, max_val=10000):
    def validator(instance, attribute, value):
        if not isinstance(value, int) or not (min_val <= value <= max_val):
            raise ValueError(f"{attribute.name} must be {min_val}–{max_val}, got {value}")
    return validator


def optional_in_choices(choices: list[str]):
    def validator(instance, attribute, value):
        if value is not None and value not in choices:
            raise ValueError(f"{attribute.name} must be one of {choices}, got {value!r}")
    return validator
```

---

## Shared Utilities

Reusable helpers live in `use_cases/_shared/` — keep use case files focused on orchestration:

| Module | Purpose |
|--------|---------|
| `command_validators.py` | Attrs validators: `non_empty_string`, `positive_int_in_range` |
| `service_resolver.py` | Typed resolver functions for external service protocols |
| `entity_resolver.py` | Lookup by ID or external identifier, raise `NotFoundError` if missing |

Pattern: if logic appears in 3+ use cases, extract to `_shared/`. If it's in 1–2, keep it inline — premature extraction is worse than repetition.

---

## Intentional Pattern Repetition

Each use case owns its own Command, Result, transaction boundary, and error handling. **Don't** extract a `BaseUseCase` or shared generic:

```python
# ❌ Don't do this — "DRY" that destroys local comprehensibility
class BaseUseCase[TCommand, TResult]:
    async def execute(self, command: TCommand, uow: UnitOfWorkProtocol) -> TResult:
        async with uow:
            result = await self._do_work(command, uow)
            await uow.commit()
            return result


# ✓ Do this — each file is self-contained and readable top-to-bottom
class CreateOrderUseCase:
    async def execute(self, command, uow):
        async with uow:
            ...
            await uow.commit()
            return result
```

Why: you can read any single use case file and understand the entire flow without chasing inheritance chains. The 5 lines of "boilerplate" (`async with uow`, `commit`, `return`) are the transaction contract — they belong in every file.

---

## Testing Use Cases

Use case tests mock the UoW and assert on Results. See [Testing Strategy](testing-strategy.md) for fixture patterns.

```python
class TestCreateOrderHappyPath:
    @pytest.mark.asyncio
    async def test_creates_order(self):
        uow = make_mock_uow()
        uow.get_order_repository().create.return_value = Order(id=1, ...)

        result = await CreateOrderUseCase().execute(
            CreateOrderCommand(customer_id="cust_1", items=[...]),
            uow,
        )

        assert result.items_processed == 3
        uow.commit.assert_called_once()


class TestCreateOrderErrors:
    @pytest.mark.asyncio
    async def test_empty_items_raises(self):
        """Command validation rejects empty items at construction."""
        with pytest.raises(ValueError):
            CreateOrderCommand(customer_id="cust_1", items=[])
```

Test structure:
1. **Arrange**: `make_mock_uow()`, configure repo return values
2. **Act**: `UseCase().execute(Command(...), uow)`
3. **Assert**: Check Result fields, verify `commit()` called (or not for read-only)

---

## Audit Checklist

Use this to verify existing use cases or review new ones.

### Structure
- [ ] File has 3 `@define` classes: Command (`frozen`), Result (`frozen`), UseCase (mutable)
- [ ] `execute()` signature: `(self, command, uow: UnitOfWorkProtocol) -> Result`
- [ ] Command fields have validators where appropriate
- [ ] Even parameterless queries have an empty Command

### Transactions
- [ ] `async with uow:` wraps all repository access
- [ ] `await uow.commit()` called once on success path
- [ ] Read-only use cases (previews, lists, gets) never call `commit()`
- [ ] No repository access outside the `async with uow:` block

### Dependencies
- [ ] No imports from infrastructure layer
- [ ] Repositories from `uow.get_*_repository()`, never constructor-injected
- [ ] External services via typed resolver functions, not raw imports
- [ ] No database session creation — the runner handles that

### Interface integration
- [ ] Route handlers are 5–10 lines of parse/build/execute/serialize
- [ ] Both CLI and API use the same use case — no duplicated logic
- [ ] All calls go through `execute_use_case()` runner (see [Backend Patterns](backend-patterns.md))

### Testing
- [ ] Test file at `tests/unit/application/use_cases/test_<name>.py`
- [ ] Uses `make_mock_uow()` — not a real database
- [ ] Happy path + at least one error case
- [ ] Asserts on Result fields, not implementation details
- [ ] Verifies `commit()` called (write) or not called (read-only)
