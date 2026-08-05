---
name: python-development
version: 1.0.0
technology: python
author: Ankur Bhatnagar
description: >
  General Python scripting and utilities skill (Python 3.11+).
  Covers PEP 8 conventions, type hints, dataclasses, error handling,
  file I/O, concurrency, logging, CLI tools, and code quality.
references:
  - references/type-system.md
  - references/concurrency.md
  - references/file-operations.md
  - references/patterns.md
  - references/examples.md
---

# Python Development Skill

This skill guides Claude to produce idiomatic, type-annotated Python 3.11+ scripts,
utilities, and libraries. Follow every rule in this document unless the **Customizing**
section overrides it.

---

## 1. When to Use This Skill

Use this skill when working on:
- Python scripts and command-line tools
- Data processing pipelines and ETL utilities
- File and I/O manipulation utilities
- Library and module code without a web framework
- Async scripts using `asyncio`
- Configuration and automation scripts

Do **not** apply to FastAPI, Django, Flask, or other web-framework projects — those require
a framework-specific skill extension.

---

## 2. Project Structure

```
project-name/
├── pyproject.toml              # Build system config (PEP 517/518)
├── .python-version             # Pin Python version (e.g. 3.12)
├── README.md
├── src/
│   └── project_name/           # Main package (src-layout)
│       ├── __init__.py
│       ├── main.py             # Entry point / CLI
│       ├── config.py           # Configuration loading
│       ├── models/             # Dataclasses, TypedDicts, Pydantic models
│       │   └── __init__.py
│       ├── services/           # Business logic modules
│       │   └── __init__.py
│       └── utils/              # Shared utilities
│           └── __init__.py
└── scripts/                    # One-off scripts not part of the package
    └── migrate_data.py
```

### `pyproject.toml`

```toml
[build-system]
requires      = ["hatchling"]
build-backend = "hatchling.build"

[project]
name            = "project-name"
version         = "0.1.0"
requires-python = ">=3.11"
dependencies    = [
    "pydantic>=2.0",
    "rich>=13.0",
    "typer>=0.12",
]

[project.scripts]
project-name = "project_name.main:app"

[tool.ruff]
target-version = "py311"
line-length    = 100

[tool.ruff.lint]
select = ["E", "W", "F", "I", "N", "UP", "B", "SIM"]

[tool.mypy]
python_version  = "3.11"
strict          = true
ignore_missing_imports = true
```

---

## 3. Python Conventions

### PEP 8 + project rules

- Line length: 100 characters (enforced by Ruff)
- Indentation: 4 spaces — never tabs
- Two blank lines between top-level definitions; one blank line between methods
- Imports in order: standard library, third-party, local — separated by blank lines
- Never use wildcard imports (`from module import *`)
- Prefer absolute imports over relative imports in scripts
- Never use `!important` equivalents — use clear specificity in logic, not hacks

### Type hints (mandatory for all public functions)

```python
from __future__ import annotations

from collections.abc import Sequence
from typing import Any


def process_orders(
    orders: Sequence[dict[str, Any]],
    *,
    dry_run: bool = False,
) -> list[str]:
    """Process a list of raw order dicts and return order IDs.

    Args:
        orders:  Raw order records from the API.
        dry_run: When True, validate only — do not persist.

    Returns:
        List of successfully processed order IDs.
    """
    results: list[str] = []
    for order in orders:
        order_id = str(order.get("id", ""))
        if not dry_run:
            results.append(order_id)
    return results
```

**Annotation rules:**
- Always annotate function parameters and return types
- Use `from __future__ import annotations` in every file for deferred evaluation
- Use `collections.abc` types (`Sequence`, `Mapping`, `Callable`, `Iterator`) — not `typing.List`
- `X | Y` union syntax (not `Union[X, Y]`) — Python 3.10+
- `X | None` (not `Optional[X]`) — clearer intent
- Never use `Any` except at true boundaries (external APIs, JSON parsing)

---

## 4. Type System

### Dataclass (preferred for pure data containers)

```python
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from enum import StrEnum


class OrderStatus(StrEnum):
    PENDING   = "pending"
    SHIPPED   = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


@dataclass(frozen=True, slots=True)
class Order:
    id:           str
    customer_id:  str
    status:       OrderStatus
    total_amount: float
    created_at:   datetime
    items:        tuple[OrderItem, ...] = field(default_factory=tuple)

    def is_active(self) -> bool:
        return self.status not in (OrderStatus.DELIVERED, OrderStatus.CANCELLED)


@dataclass(frozen=True, slots=True)
class OrderItem:
    sku:        str
    quantity:   int
    unit_price: float

    @property
    def line_total(self) -> float:
        return self.quantity * self.unit_price
```

### TypedDict (for typed dict structures — JSON API shapes)

```python
from typing import TypedDict


class OrderDict(TypedDict):
    id:           str
    customer_id:  str
    status:       str
    total_amount: float
    created_at:   str


class OrderItemDict(TypedDict):
    sku:        str
    quantity:   int
    unit_price: float
```

### Pydantic model (when validation is needed)

```python
from datetime import datetime
from pydantic import BaseModel, Field, field_validator


class CreateOrderRequest(BaseModel):
    customer_id:  str   = Field(min_length=1, max_length=100)
    total_amount: float = Field(gt=0, description="Must be positive")
    items:        list[OrderItemRequest] = Field(min_length=1)

    @field_validator("customer_id")
    @classmethod
    def strip_customer_id(cls, v: str) -> str:
        return v.strip()


class OrderItemRequest(BaseModel):
    sku:      str = Field(min_length=1, max_length=50)
    quantity: int = Field(ge=1, le=1000)
```

For Protocol, TypeVar, Generic, and advanced type patterns see `references/type-system.md`.

---

## 5. Functions & Classes

### Function conventions

```python
# Named keyword arguments for booleans and optional params
result = process_orders(orders, dry_run=True, page_size=50)

# Keyword-only after *
def fetch_orders(
    customer_id: str,
    *,
    page: int = 1,
    page_size: int = 20,
    include_cancelled: bool = False,
) -> list[Order]:
    ...

# Always return early to reduce nesting
def validate_order(order: Order) -> str | None:
    """Returns an error message if invalid, None if valid."""
    if not order.customer_id:
        return "Customer ID is required"
    if order.total_amount <= 0:
        return "Total amount must be positive"
    if not order.items:
        return "Order must have at least one item"
    return None
```

### Class conventions

```python
class OrderProcessor:
    """Processes raw API orders into domain objects."""

    def __init__(self, api_url: str, *, timeout: float = 10.0) -> None:
        self._api_url = api_url
        self._timeout = timeout
        self._session: httpx.Client | None = None

    def __enter__(self) -> OrderProcessor:
        self._session = httpx.Client(
            base_url=self._api_url,
            timeout=self._timeout,
        )
        return self

    def __exit__(self, *_: object) -> None:
        if self._session:
            self._session.close()

    def process(self, raw: OrderDict) -> Order:
        return Order(
            id=raw["id"],
            customer_id=raw["customer_id"],
            status=OrderStatus(raw["status"]),
            total_amount=raw["total_amount"],
            created_at=datetime.fromisoformat(raw["created_at"]),
        )
```

---

## 6. Error Handling

```python
# Custom exception hierarchy — always inherit from a base app exception
class AppError(Exception):
    """Base exception for all application errors."""


class NotFoundError(AppError):
    def __init__(self, resource: str, id: str) -> None:
        super().__init__(f"{resource} '{id}' not found")
        self.resource = resource
        self.id       = id


class ValidationError(AppError):
    def __init__(self, field: str, message: str) -> None:
        super().__init__(f"Validation error on '{field}': {message}")
        self.field   = field
        self.message = message


class ExternalServiceError(AppError):
    def __init__(self, service: str, cause: Exception) -> None:
        super().__init__(f"External service '{service}' failed: {cause}")
        self.__cause__ = cause
```

```python
# Exception handling patterns
import logging

logger = logging.getLogger(__name__)


def load_order(order_id: str) -> Order:
    try:
        raw = fetch_from_api(order_id)
    except httpx.TimeoutException as exc:
        raise ExternalServiceError("orders-api", exc) from exc
    except httpx.HTTPStatusError as exc:
        if exc.response.status_code == 404:
            raise NotFoundError("Order", order_id) from exc
        raise ExternalServiceError("orders-api", exc) from exc

    if raw is None:
        raise NotFoundError("Order", order_id)

    return order_from_dict(raw)


# Top-level error boundary
def main() -> None:
    try:
        run()
    except AppError as exc:
        logger.error("Application error: %s", exc)
        raise SystemExit(1) from exc
    except KeyboardInterrupt:
        logger.info("Interrupted by user")
        raise SystemExit(0)
```

**Error handling rules:**
- Always chain exceptions with `raise NewError() from original_exc`
- Never silently swallow exceptions with bare `except: pass`
- Catch specific exception types — never bare `except Exception` at call sites
- Use `logger.exception()` (not `logger.error()`) when you have an exception object to include the traceback

---

## 7. Collections & Comprehensions

```python
# List comprehension — prefer over map()/filter() for readability
active_orders = [o for o in orders if o.is_active()]

# Dict comprehension
id_to_order = {o.id: o for o in orders}

# Generator expression — use for large sequences to avoid materialising in memory
total = sum(item.line_total for o in orders for item in o.items)

# Walrus operator for combined compute + filter
processed = [result for raw in data if (result := transform(raw)) is not None]

# Avoid mutating a list while iterating — build a new list instead
valid   = [o for o in orders if is_valid(o)]
invalid = [o for o in orders if not is_valid(o)]

# Use enumerate() when you need the index
for i, order in enumerate(orders, start=1):
    print(f"{i}. {order.id}")

# Use zip() for parallel iteration
for order, status in zip(orders, statuses, strict=True):
    order.update_status(status)
```

---

## 8. Configuration

```python
# config.py — load from environment variables with defaults

from __future__ import annotations

import os
from dataclasses import dataclass


@dataclass(frozen=True)
class Config:
    api_url:        str
    api_timeout:    float
    log_level:      str
    max_retries:    int

    @classmethod
    def from_env(cls) -> Config:
        return cls(
            api_url     = os.environ["API_URL"],           # Required — raises KeyError if missing
            api_timeout = float(os.getenv("API_TIMEOUT", "10.0")),
            log_level   = os.getenv("LOG_LEVEL", "INFO").upper(),
            max_retries = int(os.getenv("MAX_RETRIES", "3")),
        )
```

```python
# Usage
config = Config.from_env()

# .env file (loaded with python-dotenv in dev)
# Never commit .env to source control
```

---

## 9. Logging

```python
# logging_config.py — configure once at startup

import logging
import sys


def configure_logging(level: str = "INFO") -> None:
    """Configure structured console logging for the application."""
    logging.basicConfig(
        level   = level,
        format  = "%(asctime)s  %(levelname)-8s  %(name)s  %(message)s",
        datefmt = "%Y-%m-%dT%H:%M:%S",
        stream  = sys.stdout,
    )

    # Silence noisy third-party loggers
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)
```

```python
# In every module — use __name__ as the logger name
import logging

logger = logging.getLogger(__name__)


def process_order(order_id: str) -> None:
    logger.info("Processing order %s", order_id)         # Use % formatting — not f-strings
    try:
        result = fetch_order(order_id)
        logger.debug("Fetched order: %s", result)
        save_order(result)
        logger.info("Order %s processed successfully", order_id)
    except ExternalServiceError:
        logger.exception("Failed to process order %s", order_id)
        raise
```

**Logging rules:**
- Use `%`-style formatting in log calls — not f-strings (lazy evaluation, avoids cost when log level is off)
- `logger.exception()` automatically appends the current traceback — use it inside `except` blocks
- Never log secrets, passwords, API keys, or PII
- Configure logging once in `main()` or the entry point — never in library code

---

## 10. CLI Tools (Typer)

```python
# main.py

from __future__ import annotations

import logging
from pathlib import Path

import typer

from project_name.config import Config
from project_name.services.order_processor import OrderProcessor

app = typer.Typer(help="Order processing CLI")
logger = logging.getLogger(__name__)


@app.command()
def process(
    input_file: Path = typer.Argument(
        ...,
        exists=True,
        readable=True,
        help="Path to the JSON file containing raw orders.",
    ),
    output_dir: Path = typer.Option(
        Path("./output"),
        "--output", "-o",
        help="Directory to write processed orders.",
    ),
    dry_run: bool = typer.Option(
        False,
        "--dry-run",
        help="Validate input without writing output.",
    ),
    verbose: bool = typer.Option(False, "--verbose", "-v"),
) -> None:
    """Process raw orders from INPUT_FILE and write results to OUTPUT_DIR."""
    log_level = "DEBUG" if verbose else "INFO"
    configure_logging(log_level)

    config = Config.from_env()
    output_dir.mkdir(parents=True, exist_ok=True)

    with OrderProcessor(config) as processor:
        try:
            processor.run(input_file, output_dir, dry_run=dry_run)
        except Exception as exc:
            logger.exception("Processing failed: %s", exc)
            raise typer.Exit(code=1) from exc

    typer.echo("Done.")


if __name__ == "__main__":
    app()
```

---

## 11. Performance

- Prefer generators over lists when you only need to iterate once
- Use `functools.lru_cache` or `functools.cache` for expensive pure functions
- Avoid repeated attribute lookups in tight loops — cache with a local variable
- Use `collections.Counter`, `collections.defaultdict`, and `heapq` from the standard library — do not re-implement
- Profile before optimising — use `cProfile` + `pstats` or `py-spy` for wall-clock profiling
- Use `pathlib.Path` for all file system operations — not `os.path`

---

## 12. Naming Conventions

| Artefact | Convention | Example |
|---|---|---|
| Module / package | `snake_case` | `order_processor.py` |
| Class | `PascalCase` | `OrderProcessor` |
| Function / method | `snake_case` | `process_order()` |
| Constant | `UPPER_SNAKE` | `MAX_RETRIES = 3` |
| Private attribute | `_single_underscore` | `self._session` |
| "Dunder" | `__double_underscore__` | `__init__`, `__enter__` |
| Type alias | `PascalCase` | `OrderList = list[Order]` |
| Enum member | `UPPER_SNAKE` | `OrderStatus.PENDING` |
| Boolean variable | `is_`, `has_`, `can_` prefix | `is_active`, `has_items` |

---

## 13. Code Quality

- Run `ruff check .` and `ruff format .` before committing
- Run `mypy src/` — fix all type errors; never use `# type: ignore` without a comment explaining why
- Never use mutable default arguments: `def f(items=[])` — use `None` and assign inside
- Always use `if __name__ == "__main__":` guard in scripts
- Use `pathlib.Path` everywhere — never `os.path.join` or string concatenation for paths
- Use `with` statements for all file, network, and database resources
- Avoid bare `except:` — always name the exception type

---

## 14. Customizing This Skill

```markdown
## Project Overrides — [Project Name]

- Python version: 3.12 (use match statements, improved error messages)
- Pydantic v1 (legacy project — not v2 syntax)
- No Typer — uses argparse directly
- Configuration: YAML config file (PyYAML) instead of environment variables
- Logging: structlog with JSON output for production
```
