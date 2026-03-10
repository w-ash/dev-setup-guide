# Domain Modeling

> **Scope**: Immutable entities with attrs, Command/Result pattern, structured failures, self-describing metrics
> **Prerequisites**: [Python 3.14+ Syntax](python-syntax.md), [Project Structure](project-structure.md)
> **Deliverables**: Domain entities, use case commands/results, failure types, summary metrics
> **Estimated effort**: M

No database or infrastructure required — these patterns apply to any Python backend.

---

## Immutable Entities with attrs

Domain entities should be **frozen** (immutable) and **slotted** (memory-efficient). This prevents accidental mutation in pipelines and makes entities safe for concurrent access.

```python
import attrs
from attrs import define, field, validators


@define(frozen=True, slots=True)
class Item:
    """Immutable domain entity with built-in validation."""

    id: int | None = None
    name: str = field(validator=validators.instance_of(str))
    tags: list[str] = field(factory=list, validator=_validate_tags)
    score: float = 0.0


def _validate_tags(
    instance: object, attribute: attrs.Attribute, value: list[str]
) -> None:
    """Custom validator — use for constraints Pydantic can't express."""
    if not all(isinstance(t, str) for t in value):
        raise TypeError("All tags must be strings")
```

**Why attrs over Pydantic for domain entities**:
- Pydantic is for *boundaries* (API input/output, config parsing). attrs is for *internal domain* models.
- `frozen=True` enforces immutability at the language level — no `.model_copy()` needed.
- `slots=True` reduces memory per-instance (significant at 10,000+ items).
- Custom validators run at construction time, catching bugs immediately.

### Immutable Updates with evolve()

Never mutate — create new instances with `attrs.evolve()`:

```python
original = Item(id=1, name="Widget", score=0.5)

# Return new instance with updated fields — original is unchanged
updated = attrs.evolve(original, score=0.9)


# Chain transformations (functional pipeline)
def boost_score(item: Item) -> Item:
    return attrs.evolve(item, score=min(item.score * 1.5, 1.0))


def add_tag(item: Item, tag: str) -> Item:
    return attrs.evolve(item, tags=[*item.tags, tag])


result = add_tag(boost_score(original), "promoted")
```

### Self-Referencing Construction

When an entity needs to derive one field from another, use `__attrs_post_init__`:

```python
@define(frozen=True, slots=True)
class Order:
    items: list[LineItem]
    total: float = field(init=False)  # Computed, not passed by caller

    def __attrs_post_init__(self) -> None:
        # Use object.__setattr__ because frozen=True
        object.__setattr__(self, "total", sum(i.price for i in self.items))
```

---

## Command / Result Pattern

Every use case gets explicit **Command** (input) and **Result** (output) objects. This creates clear contracts at the application boundary.

```python
@define(frozen=True, slots=True)
class CreateOrderCommand:
    """What the caller wants to do."""

    customer_id: str
    items: list[LineItem]
    priority: str = "normal"

    def __attrs_post_init__(self) -> None:
        """Validate constraints that span multiple fields."""
        if self.priority == "rush" and len(self.items) > 100:
            raise ValueError("Rush orders limited to 100 items")


@define(frozen=True, slots=True)
class CreateOrderResult:
    """What the caller gets back — immutable snapshot of what happened."""

    order: Order
    items_processed: int
    warnings: list[str] = field(factory=list)
    duration_ms: int = 0
```

**Design rules**:
- Commands are **frozen** — validated at construction, never modified after
- Results carry **operation metadata** (timing, warnings, counts) alongside the primary output
- Even parameterless queries get a Command for API uniformity: `class GetStatsCommand: pass`
- Adding optional fields to Command/Result never breaks existing callers

```python
@define(slots=True)
class CreateOrderUseCase:
    """Use case — always `async def execute(command, uow) -> Result`."""

    async def execute(
        self, command: CreateOrderCommand, uow: UnitOfWork
    ) -> CreateOrderResult:
        # All use cases follow this exact signature pattern
        ...
```

---

## Structured Failure Tracking

Model failures as **domain objects**, not just exceptions. This enables partial success — 80 items succeed, 20 fail, and you know *why* each one failed.

```python
from enum import Enum


class FailureReason(Enum):
    """Classify failures into actionable categories."""

    NOT_FOUND = "not_found"
    API_ERROR = "api_error"
    INVALID_DATA = "invalid_data"
    RATE_LIMITED = "rate_limited"
    AUTH_EXPIRED = "auth_expired"


@define(frozen=True, slots=True)
class ProcessingFailure:
    """Structured failure — enough context to diagnose and retry intelligently."""

    entity_id: int | str
    reason: FailureReason
    service: str
    details: str = ""
```

Use failures in results instead of throwing:

```python
@define(frozen=True, slots=True)
class BatchProcessResult:
    """Partial success is the norm — track what worked and what didn't."""

    succeeded: list[Item]
    failures: list[ProcessingFailure]

    @property
    def success_rate(self) -> float:
        total = len(self.succeeded) + len(self.failures)
        return len(self.succeeded) / total if total > 0 else 0.0
```

**Why failures as domain objects**:
- **Partial success**: batch operations don't fail entirely because one item errored
- **Intelligent retry**: infrastructure can retry `RATE_LIMITED` but skip `NOT_FOUND`
- **Diagnostics**: group failures by reason for actionable reporting
- **Testing**: assert on `reason=FailureReason.API_ERROR` — no fragile exception message matching

---

## Self-Describing Summary Metrics

When operations produce statistics, make the metrics **self-describing** — carry their own labels and format hints so the UI layer doesn't need to hardcode display logic.

```python
from typing import Literal

MetricFormat = Literal["count", "percent", "duration", "bytes"]


@define(frozen=True, slots=True)
class SummaryMetric:
    """Metric that knows how to describe itself."""

    name: str  # "items_processed"
    value: int | float  # 1250
    label: str  # "Items Processed"
    format: MetricFormat = "count"
    significance: int = 0  # Display order (lower = more prominent)
```

Build metrics in result factories:

```python
def create_import_result(data: ImportData) -> list[SummaryMetric]:
    metrics = [
        SummaryMetric(
            "imported", data.imported_count, "Imported", "count", significance=1
        ),
        SummaryMetric(
            "skipped", data.skipped_count, "Skipped", "count", significance=3
        ),
    ]
    if data.success_rate > 0:
        metrics.append(
            SummaryMetric(
                "success_rate",
                data.success_rate,
                "Success Rate",
                "percent",
                significance=5,
            )
        )
    return sorted(metrics, key=lambda m: m.significance)
```

**Why self-describing**: Adding a new metric requires *zero* UI changes. The frontend iterates `metrics`, reads `label` and `format`, and renders. No coupling between backend business logic and frontend display code.

---

## Sentinel Pattern for Partial Updates

Distinguish "not provided" from "explicitly set to None" using a sentinel value:

```python
from typing import Final


class _Unset:
    """Sentinel: 'this parameter was not provided'."""

    __slots__ = ()


UNSET: Final = _Unset()


@define(frozen=True, slots=True)
class UpdateItemCommand:
    id: int
    name: str | _Unset = UNSET  # Not provided → keep current value
    description: str | None | _Unset = UNSET  # None → clear, UNSET → keep, str → set
```

Handle in use case logic:

```python
async def execute(self, command: UpdateItemCommand, uow: UnitOfWork) -> Item:
    item = await repo.get(command.id)
    updates = {}
    if not isinstance(command.name, _Unset):
        updates["name"] = command.name
    if not isinstance(command.description, _Unset):
        updates["description"] = command.description
    return attrs.evolve(item, **updates) if updates else item
```

**Three states**: `UNSET` (preserve), `None` (clear), value (set). This is essential for PATCH-style APIs and resume-friendly operations where you only update what changed.
