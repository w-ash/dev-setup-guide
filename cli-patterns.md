# CLI Patterns: Typer + Rich

> **Scope**: Typer app structure, async bridging, command modules, error handling, interactive menus, Rich output, and progress displays
> **Prerequisites**: [Python Tooling](python-tooling.md)
> **Deliverables**: CLI app with subcommand registration, async bridge, interactive menus, styled output, progress display
> **Estimated effort**: M

---

## App Structure

### Root App with Subcommands

```python
# src/interface/cli/app.py
import typer
from typing import Annotated

app = typer.Typer(
    help="My Project CLI",
    no_args_is_help=True,
    rich_markup_mode="rich",
    add_completion=False,
    pretty_exceptions_enable=True,
)
```

### Initialization Callback

```python
@app.callback()
def init_cli(
    verbose: Annotated[bool, typer.Option("--verbose", "-v")] = False,
) -> None:
    setup_logging(verbose)
    Path("data").mkdir(exist_ok=True)
```

### Subcommand Registration

Organize commands into domain-specific modules, each with its own `Typer` app:

```python
def _register_commands() -> None:
    """Lazy imports avoid circular dependencies."""
    from src.interface.cli import item_commands, import_commands

    app.add_typer(
        item_commands.app,
        name="items",
        help="Manage items",
        rich_help_panel="Item Management",
    )
    app.add_typer(
        import_commands.app,
        name="import",
        help="Import data from external services",
        rich_help_panel="Data Management",
    )


_register_commands()
```

---

## Async Bridge

Typer commands are synchronous, but your application layer is async. Bridge the gap:

```python
# src/interface/cli/async_runner.py
import asyncio
from collections.abc import Coroutine
from concurrent.futures import ThreadPoolExecutor
from typing import Any


def run_async[T](coro: Coroutine[Any, Any, T]) -> T:
    """Bridge sync CLI -> async application layer."""

    async def _run_with_executor() -> T:
        loop = asyncio.get_running_loop()
        loop.set_default_executor(ThreadPoolExecutor(max_workers=8))
        return await coro

    return asyncio.run(_run_with_executor())
```

**Usage in commands**:
```python
@app.command()
def sync_data() -> None:
    result = run_async(execute_use_case(lambda uow: SyncDataUseCase(uow).execute()))
    display_result(result)
```

This uses the same `execute_use_case()` runner as FastAPI — zero business logic duplication. See [Backend Patterns](backend-patterns.md) for the runner.

---

## Command Modules

Each domain gets its own module with a local Typer app:

```python
# src/interface/cli/item_commands.py
import typer

app = typer.Typer(
    help="Manage items",
    no_args_is_help=False,  # Allow bare invocation for interactive mode
    rich_markup_mode="rich",
)
```

### Progressive Discovery

Use `invoke_without_command=True` to show an interactive browser when called with no subcommand:

```python
@app.callback(invoke_without_command=True)
def items_main(ctx: typer.Context) -> None:
    if ctx.invoked_subcommand is None:
        _show_interactive_list()


@app.command()
def run(item_id: str | None = None) -> None:
    """Execute a specific operation."""
    if item_id is None:
        item_id = _prompt_for_selection()
    _execute_operation(item_id)
```

Supports both interactive users (`my-app items` -> browser) and automation (`my-app items run --item-id=my-item`).

---

## Error Handling

```python
import typer
from typing import Never


def handle_cli_error(e: Exception, message: str) -> Never:
    err_console = get_err_console()
    err_console.print(f"[red]Error: {message}: {e}[/red]")
    raise typer.Exit(1) from e
```

**Usage**:
```python
@app.command()
def import_data(source: str) -> None:
    try:
        result = run_async(_do_import(source))
        display_result(result)
    except ConnectionError as e:
        handle_cli_error(e, f"Failed to connect to {source}")
```

---

## Interactive Menus

A reusable menu component eliminates Panel -> Prompt -> dispatch boilerplate:

```python
from attrs import define


@define(frozen=True, slots=True)
class MenuOption:
    key: str
    aliases: list[str]
    label: str  # Rich markup: "[bold]Service A[/bold] -- Import records"
    handler: Callable[[], None]


def run_interactive_menu(
    *,
    title: str,
    subtitle: str,
    options: list[MenuOption],
) -> None:
    console = get_console()
    console.print(Panel.fit(f"[bold]{title}[/bold]\n{subtitle}"))

    for opt in options:
        console.print(f"  [{opt.key}] {opt.label}")

    choice = Prompt.ask(
        "Select",
        choices=[o.key for o in options] + [a for o in options for a in o.aliases],
    )
    selected = next(o for o in options if choice == o.key or choice in o.aliases)
    selected.handler()
```

**Usage**:
```python
run_interactive_menu(
    title="Data Import",
    subtitle="Import your data from an external service",
    options=[
        MenuOption(
            key="1",
            aliases=["svc-a"],
            label="[bold]Service A[/bold] -- Import records",
            handler=_import_service_a,
        ),
        MenuOption(
            key="2",
            aliases=["svc-b"],
            label="[bold]Service B[/bold] -- Import records",
            handler=_import_service_b,
        ),
    ],
)
```

---

## Console Management

Global singleton consoles for consistent terminal width detection:

```python
# src/interface/cli/console.py
from rich.console import Console

_console: Console | None = None


def get_console() -> Console:
    global _console
    if _console is None:
        _console = Console()
    return _console


_err_console: Console | None = None


def get_err_console() -> Console:
    global _err_console
    if _err_console is None:
        _err_console = Console(stderr=True)
    return _err_console
```

---

## Rich Output Patterns

### Result Tables

```python
def display_result(result: OperationResult) -> None:
    table = Table(show_header=False, box=None, padding=(0, 2))
    table.add_column(style="cyan")
    table.add_column(style="green bold")
    for metric in result.metrics:
        table.add_row(metric.label, _format_value(metric.value))
    get_console().print(table)
```

### Styled Panels

```python
console.print(
    Panel.fit(
        f"[bold green]Success[/bold green] -- Processed {count} items",
        border_style="green",
    )
)
```

### Dim Styling for Secondary Data

```python
style = "bold" if item.is_fresh else "dim"
table.add_row(f"[{style}]{item.name}[/{style}]", str(item.count))
```

---

## Progress Display

For long-running operations, use Rich's `Progress` + `Live`:

```python
from rich.live import Live
from rich.progress import Progress, SpinnerColumn, BarColumn, TextColumn


async def run_with_progress(operations: list[Operation]) -> None:
    progress = Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        BarColumn(),
        TextColumn("{task.completed}/{task.total}"),
        refresh_per_second=10,
    )

    with Live(progress, refresh_per_second=10, redirect_stdout=True):
        for op in operations:
            task_id = progress.add_task(op.name, total=op.total_items)

            async for item in op.execute():
                progress.update(task_id, advance=1)

            progress.update(task_id, description=f"[green]ok[/green] {op.name}")
```

**Sub-operation indenting**:
```python
sub_task = progress.add_task(f"  -> {sub_op.name}", total=sub_op.total_items)
```

---

## Helper Utilities

### Date Parsing

```python
def parse_date_string(
    date_str: str | None, field_name: str = "date"
) -> datetime | None:
    if not date_str:
        return None
    try:
        return datetime.strptime(date_str, "%Y-%m-%d").replace(tzinfo=UTC)
    except ValueError:
        get_err_console().print(
            f"[red]Invalid {field_name} format: {date_str}. Use YYYY-MM-DD.[/red]"
        )
        raise typer.Exit(1) from None
```

### Confirmation Prompts

```python
if not typer.confirm(f"This will process {count} items. Continue?"):
    raise typer.Abort()
```

---

## Key Design Principles

- **Commands are thin** — 10-20 lines of parameter handling + delegation
- **Same use case runner** as FastAPI — `execute_use_case()` shared across CLI and web
- **Progressive discovery** — interactive menus for humans, direct flags for automation
- **Consistent Rich styling** — `[bold]`, `[cyan]`, `[dim]` for visual hierarchy
- **Proper exit codes** — `typer.Exit(1)` for errors, not bare exceptions
