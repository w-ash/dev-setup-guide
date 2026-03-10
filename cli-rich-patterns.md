# CLI Rich Patterns

> **Scope**: Interactive menus, console management, Rich output styling, progress displays, and helper utilities for Typer CLIs
> **Prerequisites**: [CLI with Typer](cli-typer.md)
> **Deliverables**: Reusable interactive menu component, styled output patterns, progress display for long operations
> **Estimated effort**: S

---

## Interactive Menus

A reusable menu component eliminates Panel → Prompt → dispatch boilerplate:

```python
from attrs import define


@define(frozen=True, slots=True)
class MenuOption:
    key: str  # "1", "2", etc.
    aliases: list[str]  # ["lastfm", "last.fm"]
    label: str  # Rich markup: "[bold]Last.fm[/bold] — Import history"
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
    subtitle="Import your listening history",
    options=[
        MenuOption(
            key="1",
            aliases=["lastfm"],
            label="[bold]Last.fm[/bold] — Import scrobbles",
            handler=_import_lastfm,
        ),
        MenuOption(
            key="2",
            aliases=["spotify"],
            label="[bold]Spotify[/bold] — Import streaming history",
            handler=_import_spotify,
        ),
    ],
)
```

---

## Console Management

A global singleton console ensures consistent terminal width detection:

```python
# src/interface/cli/console.py
from rich.console import Console

_console: Console | None = None


def get_console() -> Console:
    global _console
    if _console is None:
        _console = Console()
    return _console
```

For error output, use a separate stderr console:

```python
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
from rich.table import Table


def display_result(result: OperationResult) -> None:
    console = get_console()

    table = Table(show_header=False, box=None, padding=(0, 2))
    table.add_column(style="cyan")
    table.add_column(style="green bold")

    for metric in result.metrics:
        table.add_row(metric.label, _format_value(metric.value))

    console.print(table)
```

### Styled Panels

```python
from rich.panel import Panel

console.print(
    Panel.fit(
        f"[bold green]Success[/bold green] — Processed {count} items",
        border_style="green",
    )
)
```

### Dim Styling for Secondary Data

```python
# Fresh data in bold, cached data dimmed
style = "bold" if item.is_fresh else "dim"
table.add_row(f"[{style}]{item.name}[/{style}]", str(item.count))
```

---

## Progress Display

For long-running operations, use Rich's `Progress` + `Live` for coordinated terminal output:

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

            progress.update(task_id, description=f"[green]✓[/green] {op.name}")
```

**Sub-operation indenting**: Display child operations as `  ↳ Description` for visual hierarchy.

---

## Helper Utilities

Common patterns that appear across command modules:

### Date Parsing with Validation

```python
from datetime import UTC, datetime


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
