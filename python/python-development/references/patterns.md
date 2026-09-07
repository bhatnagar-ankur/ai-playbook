---
author: Ankur Bhatnagar
---

# Patterns — Full Reference

Decorators, context managers, functional patterns, and idiomatic Python design patterns.

---

## Table of Contents
1. [Decorators](#decorators)
2. [Context Managers](#context-managers)
3. [Iterators and Generators](#iterators-and-generators)
4. [Functional Patterns](#functional-patterns)
5. [Structural Pattern Matching](#structural-pattern-matching)
6. [Descriptor Protocol](#descriptor-protocol)
7. [Abstract Base Classes](#abstract-base-classes)
8. [Registry Pattern](#registry-pattern)

---

## Decorators

```python
from __future__ import annotations

import functools
import logging
import time
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
T = TypeVar("T")

logger = logging.getLogger(__name__)


# Timing decorator
def timed(func: Callable[P, T]) -> Callable[P, T]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
        start  = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        logger.debug("%s took %.3fs", func.__qualname__, elapsed)
        return result
    return wrapper


# Retry decorator with exponential back-off
def retry(
    *,
    attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: tuple[type[Exception], ...] = (Exception,),
) -> Callable[[Callable[P, T]], Callable[P, T]]:
    def decorator(func: Callable[P, T]) -> Callable[P, T]:
        @functools.wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
            wait = delay
            for attempt in range(1, attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    if attempt == attempts:
                        raise
                    logger.warning(
                        "%s failed (attempt %d/%d): %s — retrying in %.1fs",
                        func.__qualname__, attempt, attempts, exc, wait,
                    )
                    time.sleep(wait)
                    wait *= backoff
            raise RuntimeError("Unreachable")  # mypy appeasement
        return wrapper
    return decorator


# Usage
@retry(attempts=3, delay=0.5, exceptions=(ConnectionError, TimeoutError))
@timed
def fetch_data(url: str) -> dict[str, object]:
    ...


# Class decorator — add methods to a class
def singleton(cls: type[T]) -> type[T]:
    instances: dict[type, object] = {}

    @functools.wraps(cls)
    def get_instance(*args: object, **kwargs: object) -> object:
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance  # type: ignore[return-value]
```

---

## Context Managers

```python
from __future__ import annotations

from contextlib import contextmanager, suppress
from pathlib import Path
import tempfile


# Function-based context manager
@contextmanager
def atomic_write(path: Path, encoding: str = "utf-8"):
    """Write to a temp file, then replace atomically on success."""
    tmp = path.with_suffix(".tmp")
    try:
        with tmp.open("w", encoding=encoding) as f:
            yield f
        tmp.replace(path)
    except Exception:
        tmp.unlink(missing_ok=True)
        raise


# Usage
with atomic_write(Path("output/report.json")) as f:
    import json
    json.dump(data, f, indent=2)


# Timer context manager
@contextmanager
def timer(label: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        logger.info("%s: %.3fs", label, elapsed)


with timer("Load orders"):
    orders = load_all_orders()


# suppress — silently ignore specific exceptions
with suppress(FileNotFoundError):
    Path("optional-cache.json").unlink()


# Class-based context manager
class DatabaseTransaction:
    def __init__(self, connection) -> None:
        self._conn = connection

    def __enter__(self) -> DatabaseTransaction:
        self._conn.begin()
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: object,
    ) -> bool:
        if exc_type is None:
            self._conn.commit()
        else:
            self._conn.rollback()
        return False   # Do not suppress the exception
```

---

## Iterators and Generators

```python
from __future__ import annotations

from collections.abc import Generator, Iterator
from dataclasses import dataclass


# Generator function — lazy sequence
def fibonacci() -> Generator[int, None, None]:
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b


# Take n items from a generator
from itertools import islice

first_10 = list(islice(fibonacci(), 10))


# Custom iterator class
@dataclass
class OrderBatch:
    orders:     list[dict[str, object]]
    batch_size: int = 100

    def __iter__(self) -> Iterator[list[dict[str, object]]]:
        for i in range(0, len(self.orders), self.batch_size):
            yield self.orders[i : i + self.batch_size]


# itertools patterns
from itertools import (
    batched,        # Python 3.12+ only — split into chunks; see fallbacks below for 3.11
    chain,          # Flatten multiple iterables
    groupby,        # Group consecutive items
    takewhile,      # Take while predicate is True
    dropwhile,      # Drop while predicate is True
)

import itertools

# batched (Python 3.12+) — optional pattern; this skill's baseline is 3.11, so guard
# usage or use one of the fallbacks below if the target runtime may still be 3.11.
for chunk in itertools.batched(orders, 50):
    process_batch(list(chunk))

# Python 3.11-compatible fallback #1 — manual chunking loop, no extra dependency
def chunked(items: list[object], size: int) -> Iterator[list[object]]:
    for i in range(0, len(items), size):
        yield items[i : i + size]

for chunk in chunked(orders, 50):
    process_batch(chunk)

# Python 3.11-compatible fallback #2 — more_itertools.batched (pip install more-itertools)
from more_itertools import batched as batched_compat

for chunk in batched_compat(orders, 50):
    process_batch(list(chunk))

# chain — combine iterables without building a list
all_orders = list(itertools.chain(pending_orders, shipped_orders))

# groupby — items MUST be sorted first
sorted_orders = sorted(orders, key=lambda o: o["status"])
for status, group in itertools.groupby(sorted_orders, key=lambda o: o["status"]):
    print(f"{status}: {list(group)}")
```

---

## Functional Patterns

```python
from __future__ import annotations

import functools
from collections.abc import Callable
from typing import TypeVar

T = TypeVar("T")


# functools.cache — unlimited LRU cache (Python 3.9+)
@functools.cache
def compute_tax(amount: float, rate: float) -> float:
    return round(amount * rate, 2)


# functools.lru_cache — with size limit
@functools.lru_cache(maxsize=256)
def get_config(key: str) -> str:
    return load_config_value(key)


# functools.partial — partial application
from functools import partial

def fetch_orders(base_url: str, page: int, page_size: int) -> list[dict]:
    ...

fetch_page = partial(fetch_orders, "https://api.example.com", page_size=50)
page_1 = fetch_page(page=1)
page_2 = fetch_page(page=2)


# reduce — fold a sequence into a single value
from functools import reduce

total = reduce(lambda acc, o: acc + o["amount"], orders, 0.0)


# Compose functions
def compose(*funcs: Callable) -> Callable:
    """Right-to-left function composition: compose(f, g)(x) == f(g(x))"""
    return functools.reduce(lambda f, g: lambda x: f(g(x)), funcs)


normalise = compose(str.strip, str.lower)
normalise("  HELLO  ")  # "hello"
```

---

## Structural Pattern Matching

```python
# Python 3.10+ match statement — use instead of long if/elif chains

from __future__ import annotations


def handle_api_response(response: dict[str, object]) -> str:
    match response:
        case {"status": "ok",       "data": data}:
            return f"Success: {data}"
        case {"status": "error",    "message": str(msg)}:
            return f"Error: {msg}"
        case {"status": "redirect", "url": str(url)}:
            return f"Redirect to: {url}"
        case {"status": str(code)}:
            return f"Unknown status: {code}"
        case _:
            return "Invalid response"


def classify_order(order: dict[str, object]) -> str:
    match order:
        case {"status": "shipped",   "carrier": str(c)}:
            return f"Shipped via {c}"
        case {"status": "pending",   "items": []}:
            return "Empty pending order"
        case {"status": "pending",   "items": [*items]}:
            return f"Pending with {len(items)} items"
        case {"status": "cancelled", "reason": reason}:
            return f"Cancelled: {reason}"
        case _:
            return "Unknown"


# With dataclasses / classes
from dataclasses import dataclass


@dataclass
class Point:
    x: float
    y: float


def describe(shape: object) -> str:
    match shape:
        case Point(x=0, y=0):
            return "Origin"
        case Point(x=0, y=y):
            return f"On y-axis at {y}"
        case Point(x=x, y=0):
            return f"On x-axis at {x}"
        case Point(x=x, y=y):
            return f"Point at ({x}, {y})"
        case _:
            return "Unknown shape"
```

---

## Descriptor Protocol

```python
from __future__ import annotations


class Validated:
    """Descriptor that validates a value on assignment."""

    def __set_name__(self, owner: type, name: str) -> None:
        self._name = name

    def __get__(self, obj: object, objtype: type | None = None) -> object:
        if obj is None:
            return self
        return getattr(obj, f"_{self._name}", None)

    def __set__(self, obj: object, value: object) -> None:
        self.validate(value)
        setattr(obj, f"_{self._name}", value)

    def validate(self, value: object) -> None:
        pass   # Override in subclasses


class PositiveFloat(Validated):
    def validate(self, value: object) -> None:
        if not isinstance(value, float | int) or value <= 0:
            raise ValueError(f"{self._name} must be a positive number, got {value!r}")


class NonEmptyString(Validated):
    def validate(self, value: object) -> None:
        if not isinstance(value, str) or not value.strip():
            raise ValueError(f"{self._name} must be a non-empty string")


class Order:
    customer_id  = NonEmptyString()
    total_amount = PositiveFloat()

    def __init__(self, customer_id: str, total_amount: float) -> None:
        self.customer_id  = customer_id    # Descriptor validates here
        self.total_amount = total_amount   # Descriptor validates here
```

---

## Abstract Base Classes

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from pathlib import Path


class ReportWriter(ABC):
    """Abstract base for report serialisers."""

    @abstractmethod
    def write(self, data: list[dict], path: Path) -> None:
        """Serialise data and write to path."""
        ...

    @abstractmethod
    def extension(self) -> str:
        """File extension including dot, e.g. '.csv'."""
        ...

    def write_to_dir(self, data: list[dict], directory: Path, stem: str) -> Path:
        """Convenience method — calls write() with the correct path."""
        path = directory / f"{stem}{self.extension()}"
        self.write(data, path)
        return path


class CsvReportWriter(ReportWriter):
    def write(self, data: list[dict], path: Path) -> None:
        import csv
        with path.open("w", encoding="utf-8", newline="") as f:
            if data:
                writer = csv.DictWriter(f, fieldnames=data[0].keys())
                writer.writeheader()
                writer.writerows(data)

    def extension(self) -> str:
        return ".csv"


class JsonReportWriter(ReportWriter):
    def write(self, data: list[dict], path: Path) -> None:
        import json
        path.write_text(json.dumps(data, indent=2), encoding="utf-8")

    def extension(self) -> str:
        return ".json"
```

---

## Registry Pattern

```python
from __future__ import annotations

from collections.abc import Callable
from typing import TypeVar

T = TypeVar("T")


class HandlerRegistry:
    """Map event types to handler functions at runtime."""

    def __init__(self) -> None:
        self._handlers: dict[str, Callable] = {}

    def register(self, event_type: str) -> Callable:
        def decorator(func: Callable) -> Callable:
            self._handlers[event_type] = func
            return func
        return decorator

    def dispatch(self, event_type: str, payload: dict) -> object:
        handler = self._handlers.get(event_type)
        if handler is None:
            raise KeyError(f"No handler for event type '{event_type}'")
        return handler(payload)

    def has(self, event_type: str) -> bool:
        return event_type in self._handlers


registry = HandlerRegistry()


@registry.register("order.created")
def handle_order_created(payload: dict) -> str:
    return f"Created: {payload['id']}"


@registry.register("order.shipped")
def handle_order_shipped(payload: dict) -> str:
    return f"Shipped: {payload['id']} via {payload.get('carrier')}"


# Usage
result = registry.dispatch("order.created", {"id": "ORD-001"})
```
