# CLI with Typer

> **Scope**: Typer app structure, async bridging to application layer, command module organization, error handling, and design principles
> **Prerequisites**: [Python Tooling](python-tooling.md), [FastAPI Backend](fastapi-backend.md) (for shared use case runner)
> **Deliverables**: CLI app with subcommand registration, async bridge working, error handling in place
> **Estimated effort**: M

Interactive menus, Rich output styling, and progress displays are covered in [CLI Rich Patterns](cli-rich-patterns.md).

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
    rich_markup_mode="rich",  # Enable [bold], [cyan], etc. in help text
    add_completion=False,
    pretty_exceptions_enable=True,
)
```

### Initialization Callback

The `@app.callback()` runs before every command — use it for global setup:

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
    from src.interface.cli import history_commands, workflow_commands

    app.add_typer(
        workflow_commands.app,
        name="workflow",
        help="Execute and manage workflows",
        rich_help_panel="Workflow Execution",
    )
    app.add_typer(
        history_commands.app,
        name="history",
        help="Import and manage history",
        rich_help_panel="Data Management",
    )


_register_commands()
```

---

## Async Bridge

Typer commands are synchronous, but your application layer is async. Bridge the gap with a simple runner:

```python
# src/interface/cli/async_runner.py
import asyncio
from collections.abc import Coroutine
from concurrent.futures import ThreadPoolExecutor
from typing import Any


def run_async[T](coro: Coroutine[Any, Any, T]) -> T:
    """Bridge sync CLI → async application layer."""

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

This uses the same `execute_use_case()` runner as FastAPI — zero business logic duplication between CLI and web. See [FastAPI Backend](fastapi-backend.md) for the runner pattern.

---

## Command Modules

Each domain gets its own module with a local Typer app:

```python
# src/interface/cli/workflow_commands.py
import typer

app = typer.Typer(
    help="Execute and manage workflows",
    no_args_is_help=False,  # Allow bare invocation for interactive mode
    rich_markup_mode="rich",
)
```

### Progressive Discovery

Use `invoke_without_command=True` to show an interactive browser when called with no subcommand:

```python
@app.callback(invoke_without_command=True)
def workflow_main(ctx: typer.Context) -> None:
    if ctx.invoked_subcommand is None:
        _show_interactive_browser()


@app.command()
def run(workflow_id: str | None = None) -> None:
    """Execute a specific workflow."""
    if workflow_id is None:
        workflow_id = _prompt_for_selection()
    _execute_workflow(workflow_id)
```

This supports both interactive users (`my-app workflow` → browser) and automation (`my-app workflow run --workflow-id=my-flow`).

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

**Usage in commands**:
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

## Key Design Principles

- **Commands are thin** — 10-20 lines of parameter handling + delegation
- **Same use case runner** as FastAPI — `execute_use_case()` shared across CLI and web
- **Progressive discovery** — interactive menus for humans, direct flags for automation
- **Consistent Rich styling** — `[bold]`, `[cyan]`, `[dim]` for visual hierarchy
- **Proper exit codes** — `typer.Exit(1)` for errors, not bare exceptions
