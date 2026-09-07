---
author: Ankur Bhatnagar
---

# Type System — Full Reference

Dataclasses, TypedDict, Protocol, TypeVar, Generic, and runtime validation patterns
for Python 3.11+.
For the overview see the **Type System** section in `SKILL.md`.

---

## Table of Contents
1. [Dataclasses](#dataclasses)
2. [TypedDict](#typeddict)
3. [NamedTuple](#namedtuple)
4. [Protocol (structural typing)](#protocol-structural-typing)
5. [TypeVar and Generic](#typevar-and-generic)
6. [Literal and Final](#literal-and-final)
7. [Annotated and custom validators](#annotated-and-custom-validators)
8. [Pydantic v2 Patterns](#pydantic-v2-patterns)
9. [Type Guards](#type-guards)

---

## Dataclasses

```python
from __future__ import annotations

from dataclasses import dataclass, field, KW_ONLY
from datetime import datetime


# frozen=True — immutable (use for value objects)
# slots=True  — lower memory, faster attribute access (Python 3.10+)
@dataclass(frozen=True, slots=True)
class Money:
    amount:   float
    currency: str = "GBP"

    def __post_init__(self) -> None:
        if self.amount < 0:
            raise ValueError(f"Amount cannot be negative: {self.amount}")

    def add(self, other: Money) -> Money:
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)


# Mutable dataclass with defaults
@dataclass
class OrderBuilder:
    customer_id: str
    _: KW_ONLY              # Everything after is keyword-only
    status:      str   = "pending"
    items:       list[OrderItem] = field(default_factory=list)
    metadata:    dict[str, str]  = field(default_factory=dict)

    def add_item(self, item: OrderItem) -> None:
        self.items.append(item)

    def build(self) -> Order:
        if not self.items:
            raise ValueError("Order must have at least one item")
        return Order(
            customer_id = self.customer_id,
            status      = self.status,
            items       = tuple(self.items),
        )


# field() patterns
@dataclass
class Config:
    api_url:     str
    timeout:     float = field(default=10.0)
    # repr=False — exclude from __repr__ (passwords, keys)
    api_key:     str   = field(default="", repr=False)
    # compare=False — exclude from __eq__/__lt__ comparisons
    created_at:  datetime = field(
        default_factory=datetime.utcnow,
        compare=False,
    )
    # init=False — not a constructor parameter
    _cache:      dict[str, object] = field(
        default_factory=dict,
        init=False,
        repr=False,
    )
```

---

## TypedDict

```python
from __future__ import annotations

from typing import NotRequired, Required, TypedDict


# All keys required (default)
class OrderDict(TypedDict):
    id:           str
    customer_id:  str
    status:       str
    total_amount: float
    created_at:   str


# Mixed required / optional with total=False
class OrderPatchDict(TypedDict, total=False):
    status:  str
    address: str


# Explicit Required/NotRequired (PEP 655)
class OrderCreateDict(TypedDict):
    customer_id:  Required[str]
    total_amount: Required[float]
    reference:    NotRequired[str]  # May be absent


# Nested
class ApiResponseDict(TypedDict):
    data:   list[OrderDict]
    total:  int
    page:   int
    errors: NotRequired[list[str]]
```

---

## NamedTuple

```python
from __future__ import annotations

from typing import NamedTuple


class Coordinate(NamedTuple):
    lat: float
    lng: float
    alt: float = 0.0

    def distance_to(self, other: Coordinate) -> float:
        return ((self.lat - other.lat)**2 + (self.lng - other.lng)**2) ** 0.5


# NamedTuple vs dataclass:
# - NamedTuple is a tuple subclass — supports unpacking, indexing, comparison
# - dataclass is more flexible — supports __post_init__, inheritance, factories
# - Use NamedTuple for simple value objects that behave like tuples
# - Use dataclass(frozen=True) for everything else
```

---

## Protocol (structural typing)

```python
from __future__ import annotations

from typing import Protocol, TypeVar, runtime_checkable


# Protocol defines an interface without inheritance
class Serialisable(Protocol):
    def to_dict(self) -> dict[str, object]: ...


class Loggable(Protocol):
    @property
    def log_message(self) -> str: ...


# runtime_checkable — allows isinstance() checks
@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...


# Any class that implements these methods satisfies the protocol
class OrderReport:
    def to_dict(self) -> dict[str, object]:
        return {"type": "report"}

    @property
    def log_message(self) -> str:
        return f"OrderReport"


# Works without OrderReport inheriting Serialisable
def serialise(obj: Serialisable) -> str:
    import json
    return json.dumps(obj.to_dict())

serialise(OrderReport())  # Type-safe


# Protocol with state
# Python 3.12+ syntax (PEP 695 generic class) — requires Python 3.12+, not 3.11
class Repository[T](Protocol):
    def get_by_id(self, id: str) -> T | None: ...
    def save(self, entity: T) -> None: ...
    def delete(self, id: str) -> None: ...


# Python 3.11-compatible equivalent using TypeVar
RepoT = TypeVar("RepoT")


class RepositoryLegacy(Protocol[RepoT]):
    def get_by_id(self, id: str) -> RepoT | None: ...
    def save(self, entity: RepoT) -> None: ...
    def delete(self, id: str) -> None: ...
```

---

## TypeVar and Generic

```python
from __future__ import annotations

from typing import TypeVar, Generic


T  = TypeVar("T")
T_co = TypeVar("T_co", covariant=True)  # Read-only containers


# Generic function
def first_or_default(items: list[T], default: T) -> T:
    return items[0] if items else default


# Generic class
class Result(Generic[T]):
    """Represents either a successful value or an error."""

    def __init__(
        self,
        *,
        value: T | None = None,
        error: Exception | None = None,
    ) -> None:
        self._value = value
        self._error = error

    @classmethod
    def ok(cls, value: T) -> Result[T]:
        return cls(value=value)

    @classmethod
    def fail(cls, error: Exception) -> Result[T]:
        return cls(error=error)

    @property
    def is_ok(self) -> bool:
        return self._error is None

    @property
    def value(self) -> T:
        if self._error is not None:
            raise self._error
        return self._value  # type: ignore[return-value]

    @property
    def error(self) -> Exception | None:
        return self._error


# Python 3.12+ syntax (PEP 695 — no TypeVar needed)
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("Stack is empty")
        return self._items.pop()
```

---

## Literal and Final

```python
from __future__ import annotations

from typing import Final, Literal


# Final — variable cannot be reassigned
MAX_RETRIES: Final = 3
BASE_URL:    Final[str] = "https://api.example.com"


# Literal — one of a fixed set of values
def set_status(status: Literal["pending", "shipped", "delivered"]) -> None:
    ...


OrderSortField = Literal["created_at", "total_amount", "customer_id"]


def list_orders(
    sort_by: OrderSortField = "created_at",
    order:   Literal["asc", "desc"] = "desc",
) -> list[Order]:
    ...
```

---

## Annotated and custom validators

```python
from __future__ import annotations

from typing import Annotated
from dataclasses import dataclass


# Annotated[T, metadata] — attach metadata (validators, docs) to a type
PositiveFloat = Annotated[float, "must be > 0"]
NonEmptyStr   = Annotated[str,   "must not be empty"]


# Use with Pydantic for automatic validation
from pydantic import BaseModel, Field


PositiveAmount = Annotated[float, Field(gt=0, description="Must be positive")]
ShortStr       = Annotated[str,   Field(min_length=1, max_length=100)]


class CreateOrderRequest(BaseModel):
    customer_id:  ShortStr
    total_amount: PositiveAmount
```

---

## Pydantic v2 Patterns

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum
from pydantic import (
    BaseModel,
    ConfigDict,
    Field,
    field_validator,
    model_validator,
)


class OrderStatus(StrEnum):
    PENDING   = "pending"
    SHIPPED   = "shipped"
    CANCELLED = "cancelled"


class OrderItem(BaseModel):
    model_config = ConfigDict(frozen=True)

    sku:       str   = Field(min_length=1, max_length=50)
    quantity:  int   = Field(ge=1, le=1000)
    unit_price: float = Field(ge=0)

    @property
    def line_total(self) -> float:
        return self.quantity * self.unit_price


class Order(BaseModel):
    model_config = ConfigDict(
        frozen=True,
        populate_by_name=True,  # Accept both alias and field name
        use_enum_values=True,
    )

    id:           str         = Field(alias="orderId")
    customer_id:  str         = Field(min_length=1)
    status:       OrderStatus = OrderStatus.PENDING
    total_amount: float       = Field(gt=0)
    items:        list[OrderItem] = Field(default_factory=list)
    created_at:   datetime

    @field_validator("customer_id")
    @classmethod
    def normalise_customer_id(cls, v: str) -> str:
        return v.strip().lower()

    @model_validator(mode="after")
    def validate_total_matches_items(self) -> Order:
        if self.items:
            expected = sum(i.line_total for i in self.items)
            if abs(self.total_amount - expected) > 0.01:
                raise ValueError(
                    f"total_amount {self.total_amount} does not match "
                    f"sum of items {expected}"
                )
        return self


# Parsing from raw JSON / dict
raw = {"orderId": "123", "customer_id": "cust-1", "total_amount": 50.0, ...}
order = Order.model_validate(raw)

# Serialise — exclude unset, use alias
payload = order.model_dump(by_alias=True, exclude_unset=True)
json_str = order.model_dump_json()
```

---

## Type Guards

```python
from __future__ import annotations

from typing import TypeGuard, TypeIs   # TypeIs is Python 3.13+


def is_order_dict(obj: object) -> TypeGuard[OrderDict]:
    return (
        isinstance(obj, dict)
        and isinstance(obj.get("id"), str)
        and isinstance(obj.get("customer_id"), str)
        and isinstance(obj.get("total_amount"), float | int)
    )


def process_unknown(data: object) -> None:
    if is_order_dict(data):
        # data is narrowed to OrderDict here
        process_order(data)
    else:
        raise ValueError(f"Unexpected data shape: {type(data)}")
```
