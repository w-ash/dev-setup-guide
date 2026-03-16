# Testing Strategy

> **Scope**: Test pyramid ratios, placement rules by layer, factory fixtures, auto-markers, and coverage targets
> **Prerequisites**: [Python Tooling](python-tooling.md) and/or [React API & Testing](react-api-testing.md)
> **Deliverables**: Test directory structure mirroring source, factory fixtures, conftest.py with auto-markers
> **Estimated effort**: S

---

## Test Pyramid

```
         /   E2E   \           Playwright (Chromium desktop)
        /------------\
       / Integration  \       pytest (real DB), Vitest (MSW)
      /----------------\
     /    Unit Tests    \     pytest (pure logic), Vitest (components)
    /--------------------\
```

---

## Backend Test Placement

| Source Layer | Test Location | Type | Key Tool |
|---|---|---|---|
| `src/domain/` | `tests/unit/domain/` | unit | No mocks needed |
| `src/application/use_cases/` | `tests/unit/application/use_cases/` | unit | Mock UoW + repos |
| `src/infrastructure/persistence/` | `tests/integration/repositories/` | integration | Real DB session |
| `src/interface/api/` | `tests/integration/api/` | integration | httpx AsyncClient |

---

## Frontend Test Placement

| Source | Test Location | Type |
|---|---|---|
| `web/src/components/` | Co-located `*.test.tsx` | Vitest + RTL |
| `web/src/hooks/` | Co-located `*.test.ts` | Vitest |
| `web/src/pages/` | Co-located `*.test.tsx` | Vitest + MSW |
| Critical user flows | `web/e2e/*.spec.ts` | Playwright |

---

## Factory Fixtures

```python
# tests/fixtures/factories.py
def make_item(*, id: int = 1, name: str = "Test Item", **overrides) -> Item:
    """Factory with keyword overrides for any field."""
    return Item(id=id, name=name, **overrides)


def make_items(count: int = 5) -> list[Item]:
    """Batch factory producing numbered items."""
    return [make_item(id=i, name=f"Item {i}") for i in range(1, count + 1)]
```

```python
# tests/fixtures/mocks.py
def make_mock_uow() -> AsyncMock:
    """Pre-wired UoW mock with all repositories."""
    uow = AsyncMock(spec=UnitOfWorkProtocol)
    uow.get_item_repository.return_value = AsyncMock(spec=ItemRepositoryProtocol)
    return uow
```

**Key patterns**:
- Factories are **functions, not fixtures** — call `make_item(title="X")` anywhere, no fixture injection
- Keyword-only args with sensible defaults — override only what your test cares about
- Batch factories produce numbered items for collection tests
- UoW mocks pre-wire all repository accessors — tests just configure return values

### Composable Mock Builder Pattern

The `make_mock_uow()` pattern eliminates the "fixture pyramid of doom" — deeply nested fixtures like `mock_uow_with_custom_track_repo_and_playlist_v2`.

```python
# tests/fixtures/mocks.py
def make_mock_repo(**overrides) -> AsyncMock:
    """Build a mock repo with sensible defaults. Override what you need."""
    repo = AsyncMock()
    repo.find_by_ids.return_value = overrides.pop("find_by_ids", {})
    repo.save_batch.return_value = overrides.pop("save_batch", [])
    for k, v in overrides.items():
        setattr(repo, k, v)
    return repo


def make_mock_uow(**repo_overrides) -> MagicMock:
    """Pre-wired UoW — all repos have defaults, override only what matters."""
    uow = MagicMock()
    uow.get_item_repository = MagicMock(
        return_value=repo_overrides.get("item_repo", make_mock_repo())
    )
    # Async context manager protocol
    uow.__aenter__ = AsyncMock(return_value=uow)
    uow.__aexit__ = AsyncMock(return_value=None)
    uow.commit = AsyncMock()
    return uow
```

Three usage levels, from simple to specific:

```python
# Level 1: All defaults — good for 80% of tests
uow = make_mock_uow()

# Level 2: Custom repo for this test
uow = make_mock_uow(item_repo=my_custom_repo)

# Level 3: Fine-tune a default repo's return value
uow = make_mock_uow()
uow.get_item_repository.return_value.find_by_ids.return_value = {1: item}
```

---

## Auto-Markers via conftest.py

```python
# tests/conftest.py
import pytest


def pytest_collection_modifyitems(items: list[pytest.Item]) -> None:
    """Auto-apply unit/integration markers based on test file location."""
    for item in items:
        path = str(item.fspath)
        if "/tests/unit/" in path:
            item.add_marker(pytest.mark.unit)
        elif "/tests/integration/" in path:
            item.add_marker(pytest.mark.integration)
```

This eliminates per-function `@pytest.mark.unit` decorators and makes `-m "unit"` / `-m "integration"` filtering reliable.

---

## Per-Test Database Isolation

Integration tests need a real database, but tests must not share state. The pattern: **unique temporary database per test, auto-schema, guaranteed cleanup**.

```python
# tests/conftest.py
import os
import tempfile
from uuid import uuid4


def pytest_configure(config: pytest.Config) -> None:
    """Safety guard: prevent tests from accidentally using production database."""
    os.environ["DATABASE_URL"] = "sqlite+aiosqlite:///tmp/pytest_guard_DO_NOT_USE.db"


@pytest.fixture
async def db_session() -> AsyncGenerator[AsyncSession]:
    """Isolated database session — unique temp DB per test."""
    db_file = f"{tempfile.gettempdir()}/test_{uuid4().hex}.db"
    original_url = os.environ.get("DATABASE_URL")
    os.environ["DATABASE_URL"] = f"sqlite+aiosqlite:///{db_file}"

    reset_engine_cache()  # Clear cached engine so a fresh one uses the new URL
    await init_db()       # Run migrations to create schema

    session = get_session_factory()()
    try:
        yield session
    finally:
        await session.rollback()
        await session.close()
        if original_url:
            os.environ["DATABASE_URL"] = original_url
        pathlib.Path(db_file).unlink(missing_ok=True)
```

**Why unique temp files**: In-memory SQLite databases can't be shared across async connections reliably. File-backed DBs with UUID names guarantee zero test interference even under `pytest-xdist` parallel execution.

**Why `reset_engine_cache()`**: SQLAlchemy caches the engine by default. Without resetting, the second test reuses the first test's engine (pointing to the first test's DB file). This causes mysterious cross-test contamination.

---

## Patching slots=True Classes

attrs classes with `@define(slots=True)` block instance-level attribute assignment, which breaks the normal `patch.object(instance, ...)` pattern:

```python
# ❌ Fails: "attribute is read-only"
with patch.object(my_instance, "__attrs_post_init__"):
    ...

# ✓ Works: patch at the CLASS level
with patch.object(MyClass, "__attrs_post_init__"):
    instance = MyClass()
```

This commonly appears in test fixtures for infrastructure classes:

```python
@pytest.fixture
def api_client() -> MyAPIClient:
    """MyAPIClient uses slots=True — patch class, not instance."""
    with patch.object(MyAPIClient, "__attrs_post_init__"):
        client = MyAPIClient()
    client._retry_policy = passthrough_retry  # Configure after construction
    return client
```

---

## Coverage Targets

| Layer | Target |
|---|---|
| Domain + Application | 85% |
| Backend overall | 80% |
| Frontend components | 60% |
| E2E critical flows | 100% of identified flows |

---

## Test Execution Strategy

Running the right scope at the right time keeps feedback tight without sacrificing confidence.

### During implementation — targeted tests ONLY

Run only the tests for the code you changed. Stop on first failure to fix immediately.

```bash
# Backend: run the specific test file
uv run pytest tests/unit/domain/test_entities.py -x

# Iterate on a single failing test
uv run pytest tests/unit/domain/test_entities.py -x -k "test_valid_creation"

# Rerun only previously-failed tests
uv run pytest --lf

# Frontend: run the specific test file
pnpm --prefix web test src/components/ItemCard.test.tsx
```

### Before committing — full fast suite

Catch cross-cutting regressions before they hit version control.

```bash
uv run pytest                    # Excludes slow/diagnostic markers
pnpm --prefix web test               # All frontend tests
```

### Full verification — version bumps, dep updates, or on request

```bash
uv run pytest -m ""              # All tests including slow
uv run basedpyright src/         # Type check
uv run ruff check .              # Lint
pnpm --prefix web check && pnpm --prefix web build
```

### Anti-patterns

- Running the full test suite after every small edit — breaks the feedback loop
- Running `basedpyright src/` after editing a single file — type-check at commit time
- Running `pnpm --prefix web build` to verify a component change — that's what `test` is for
- Running slow/diagnostic tests during normal development

Include these test execution rules in your CLAUDE.md Testing section — see [Claude Code Setup](claude-code-setup.md) for the full CLAUDE.md template.
