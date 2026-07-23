---
name: python
description: Use when working with Python code, .py files, requirements.txt, pyproject.toml, or Python projects. Covers conventions, patterns, preferred libraries, and tooling for Python development.
---

# Python Conventions

## Code Style & Naming

| Symbol Kind | Style | Example |
|---|---|---|
| Class | PascalCase | `OrderProcessor` |
| Function, method, variable | snake_case | `process_order`, `order_count` |
| Private helper (module-level) | `_` prefix | `_validate_input` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Module name | snake_case | `order_processing.py` |
| Package name | short lowercase, no underscores | `orderproc` |
| Type variable | PascalCase | `T`, `ItemType` |
| Enum member | UPPER_CASE | `Status.ACTIVE` |

- Follow PEP 8: 4-space indents, 88-character line limit (Black default), two blank lines between top-level definitions
- Use type hints for all public APIs (`def process(data: list[str]) -> int:`)
- Prefer `pathlib.Path` over `os.path` for filesystem operations
- Use `from __future__ import annotations` at the top of every file for PEP 604-style union types
- Sort imports: standard library → third-party → local, separated by blank lines; use `isort` or Ruff's `I` rule

## Project Structure

```
my-project/
├── pyproject.toml              # project metadata, dependencies, tool config
├── README.md
├── src/
│   └── my_project/             # package directory (snake_case)
│       ├── __init__.py
│       ├── cli.py              # CLI entry points
│       ├── config.py
│       ├── models.py
│       └── services/
│           ├── __init__.py
│           └── order_service.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # shared fixtures
│   ├── test_models.py
│   └── test_order_service.py
└── .python-version             # pyenv version file
```

- Use `src/` layout to prevent import confusion; `pip install -e .` in dev
- One major class or set of related functions per file
- Keep `__init__.py` minimal (re-exports only, no logic)
- Use `if __name__ == "__main__":` only in entry point scripts, not in library modules

## Type Annotations

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import assert_never


@dataclass(frozen=True)
class Order:
    id: str
    items: list[OrderItem]
    total: float


def process_orders(orders: list[Order]) -> dict[str, float]:
    """Process orders and return a dict of order_id -> total."""
    return {o.id: o.total for o in orders}
```

- Use built-in generics (`list[str]` not `List[str]`) — Python 3.9+
- Use `|` for unions (`str | None` not `Optional[str]`) — Python 3.10+
- Use `dataclasses` or `pydantic` for data containers
- Use `TypedDict` for dicts with fixed keys
- Use `Protocol` for structural subtyping (duck typing)
- Mark functions returning `None` explicitly: `def log(msg: str) -> None:`
- Use `assert_never` for exhaustiveness checking in match/if-else chains

## Error Handling

```python
class OrderError(Exception):
    """Base exception for order processing errors."""

    def __init__(self, message: str, order_id: str | None = None) -> None:
        self.order_id = order_id
        super().__init__(message)


class OrderNotFoundError(OrderError):
    """Raised when an order does not exist."""
```

- Define domain-specific exception hierarchies; inherit from a project-specific base
- Prefer specific exception types over generic `Exception` or `RuntimeError`
- Use `try`/`except` narrowly — wrap only the fallible line, not large blocks
- Use `contextlib.suppress` for intentionally ignored exceptions
- Use `raise ... from exc` for exception chaining
- Avoid bare `except:` (catches `KeyboardInterrupt`, `SystemExit`)

## Patterns & Design

### CLI Applications

- Use `click` or `typer` for CLI argument parsing
- Decorate entry points with `@app.command()` and use type hints for automatic casting
- Follow the `cli-output-contract` skill for `--quiet`/`--json` modes

```python
import typer

app = typer.Typer()


@app.command()
def process(input_path: str = typer.Option(..., "--input", "-i"),
            output_path: str = typer.Option(..., "--output", "-o"),
            quiet: bool = typer.Option(False, "--quiet", "-q")) -> None:
    """Process data from INPUT to OUTPUT."""
    ...
```

### Async

- Use `asyncio` for I/O-bound concurrency
- Use `async with` for resource management
- Use `asyncio.gather` for concurrent tasks, `asyncio.create_task` for fire-and-forget
- Prefer `httpx` over `requests` for async HTTP
- Use `anyio` for structured concurrency when you need cancellation scopes

### Data Processing

```python
import csv
from pathlib import Path


def load_csv(path: Path) -> list[dict[str, str]]:
    with path.open(newline="") as f:
        return list(csv.DictReader(f))
```

- Prefer `polars` over `pandas` for new data processing code (faster, cleaner API)
- Use `duckdb` Python API for analytical SQL queries on DataFrames/CSV/Parquet
- Use `openpyxl` for Excel `.xlsx` read/write (or `calamine` for faster read-only)
- Stream large datasets with generators / `yield` rather than loading entirely in memory
- Parameterize all SQL queries — never use f-strings for SQL

## Testing

```python
# tests/test_order_service.py
from my_project.services.order_service import process_orders


def test_process_orders_empty() -> None:
    result = process_orders([])
    assert result == {}


def test_process_orders_single() -> None:
    orders = [Order(id="1", items=[], total=10.0)]
    result = process_orders(orders)
    assert result == {"1": 10.0}
```

- Use `pytest` as the test framework
- Use `pytest.fixture` for reusable test data and resources
- Name test files `test_*.py`; name test functions `test_*`
- Use `pytest.mark.parametrize` for data-driven tests
- Use `pytest.approx` for float comparisons
- Use `pytest.raises` for expected exceptions
- Use `pytest-mock` or `unittest.mock` for mocking; prefer `monkeypatch` fixture for simple cases
- Use `pytest-cov` for coverage reporting
- Use `tmp_path` fixture for temporary filesystem operations

## Preferred Libraries

| Category | Library | Notes |
|---|---|---|
| CLI | `typer` or `click` | Typer preferred for type-hint-native CLIs |
| Async HTTP | `httpx` | Both sync and async APIs |
| Data | `polars` | Fast DataFrame library, lazy evaluation |
| SQL | `duckdb` | Embedded analytical database, great for Parquet/CSV |
| Excel | `openpyxl` | Full read/write for `.xlsx` |
| Serialization | `pydantic` | Validation + serialization |
| CLI output | `rich` | Pretty terminal output (only when not `--quiet`/`--json`) |
| Testing | `pytest` | Standard test framework |
| Mocking | `pytest-mock` | Thin wrapper over `unittest.mock` |
| Linting | `ruff` | Fast all-in-one linter + formatter |
| Type checking | `mypy` | Strict mode preferred |

## Tooling

| Command | Purpose |
|---|---|
| `ruff check .` | Lint all files |
| `ruff format .` | Format all files |
| `mypy src/` | Type-check |
| `pytest` | Run tests |
| `pytest -v --tb=short` | Verbose, short tracebacks |
| `pip install -e .` | Editable install for development |
| `uv pip install -r pyproject.toml` | Fast dependency install via `uv` |

## pyproject.toml Conventions

```toml
[build-system]
requires = ["setuptools>=75"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "my-project"
version = "0.1.0"
description = "A short description"
requires-python = ">=3.12"
dependencies = [
    "typer>=0.12",
    "polars>=1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8",
    "pytest-mock>=3",
    "ruff>=0.6",
    "mypy>=1.11",
]

[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "SIM", "ARG", "RUF"]

[tool.mypy]
strict = true
python_version = "3.12"
```

- Pin `requires-python` minimum to the project's actual minimum supported version
- Use `[project.optional-dependencies]` for dev/test/lint groups
- Apply Ruff strict linting; use `mypy --strict` for type safety
- Prefer `uv` over `pip` for faster dependency resolution

## Anti-Patterns (Avoid)

- Wildcard imports (`from module import *`)
- Mutable default arguments (`def foo(x=[])` → use `def foo(x: list | None = None)`)
- Bare `except:` (use `except Exception:` at minimum)
- `is` for value comparison (use `==`)
- Using `type()` to check types (use `isinstance()`)
- Long methods (>50 lines); break into smaller functions
- Catching `Exception` and not re-raising or logging
- Using `print()` for production logging (use the `logging` module or `rich` console)
- Hardcoded secrets, API keys, or connection strings in source code
