# Modern Python (3.10+)

The language features that, used together, make Python code feel current. If you're writing Python in 2026 and you're not using these, you're writing 2018 Python.

---

## Type-hint syntax

### `X | None` — not `Optional[X]`

```python
# OLD
from typing import Optional, Union
def find(id: int) -> Optional[User]: ...
def parse(x: Union[int, str]) -> bool: ...

# NEW (3.10+)
def find(id: int) -> User | None: ...
def parse(x: int | str) -> bool: ...
```

### `list[X]`, `dict[K, V]` — not `List[X]`, `Dict[K, V]`

```python
# OLD
from typing import List, Dict, Tuple
def f(items: List[int]) -> Dict[str, Tuple[int, str]]: ...

# NEW (3.9+)
def f(items: list[int]) -> dict[str, tuple[int, str]]: ...
```

`typing.List/Dict/Tuple/Set` are gone. Use the lowercase builtins.

### PEP 695 generic syntax — not `TypeVar`

```python
# OLD
from typing import TypeVar
T = TypeVar("T")
def first(items: list[T]) -> T | None: ...

# NEW (3.12+)
def first[T](items: list[T]) -> T | None: ...

# Classes
class Repository[T](ABC):
    def get(self, id: int) -> T: ...

# Bounded
def feed[A: Animal](animal: A) -> A: ...

# Constrained
def add[N: (int, float, Decimal)](a: N, b: N) -> N: ...
```

### `type` statement for aliases (3.12+)

```python
type Vector = list[float]
type Matrix = list[Vector]
type Json = dict[str, "Json"] | list["Json"] | str | int | float | bool | None
```

Replaces `Vector = list[float]` (which is just a variable assignment, not a real alias from the type checker's perspective).

### `Self` for fluent return types (3.11+)

```python
from typing import Self

class Builder:
    def with_x(self, x: int) -> Self:
        self._x = x
        return self
```

Subclass-safe. `-> Builder` would lock in the base type.

### `Literal` for fixed string/int sets you can't `Enum`

```python
from typing import Literal

def set_log_level(level: Literal["debug", "info", "warning", "error"]) -> None: ...
```

### `Final` for constants

```python
from typing import Final

MAX_RETRIES: Final = 3
DEFAULT_TIMEOUT: Final[float] = 30.0
```

The checker flags rebinding.

### `assert_never` for exhaustiveness

```python
from typing import assert_never

def describe(status: OrderStatus) -> str:
    match status:
        case OrderStatus.PAID: return "paid"
        case OrderStatus.PENDING: return "pending"
        case OrderStatus.REFUNDED: return "refunded"
        case _: assert_never(status)   # type checker errors if you add a new variant
```

If you ever add `OrderStatus.CANCELLED` and forget to handle it, the type checker catches it at this line.

---

## Pattern matching (`match`)

Use `match` for shape-based dispatch — particularly with dataclasses, enums, and tuples.

```python
match shape:
    case Circle(radius=r):
        return math.pi * r ** 2
    case Rectangle(width=w, height=h):
        return w * h
    case Triangle(base=b, height=h):
        return 0.5 * b * h
    case _:
        raise ValueError(f"unknown shape: {shape}")
```

Class patterns work directly on dataclasses. Tuple patterns work on `tuple`s. Dict patterns work on dicts.

```python
# Tuples
match point:
    case (0, 0): print("origin")
    case (0, y): print(f"on y-axis at {y}")
    case (x, 0): print(f"on x-axis at {x}")
    case (x, y): print(f"at {x},{y}")

# Dicts
match payload:
    case {"type": "click", "x": x, "y": y}: ...
    case {"type": "key", "code": code}: ...

# Guards
match user:
    case User(age=age) if age < 18: print("minor")
    case User(): print("adult")
```

When to use `match` vs `if/elif`:
- **Match on shape/structure → use `match`.**
- **Match on a single condition → use `if`.**
- **Avoid `match` for simple value equality** (`if x == 1: ... elif x == 2: ...` — `match` adds nothing).

---

## Walrus operator (`:=`)

Assign-and-test in one expression. Reduces variable noise in get-or-fail patterns.

```python
# Without walrus
result = compute()
if result is not None:
    process(result)

# With walrus
if (result := compute()) is not None:
    process(result)

# In a while loop
while (chunk := file.read(4096)):
    process(chunk)

# In a comprehension (use sparingly — can be obscure)
filtered = [y for x in xs if (y := transform(x)) is not None]
```

Don't overuse. If the assignment doesn't immediately participate in the condition, keep it on its own line.

---

## String methods

### `removeprefix` / `removesuffix` (3.9+)

```python
# OLD
if s.startswith("Mr. "):
    s = s[len("Mr. "):]
# Or worse:
s = s.lstrip("Mr. ")   # WRONG — lstrip removes any of those characters

# NEW
s = s.removeprefix("Mr. ")
filename = path.removesuffix(".csv")
```

### f-strings with `=` for debug printing (3.8+)

```python
x, y = 3, 4
print(f"{x=}, {y=}")        # x=3, y=4
print(f"{x*y=}")            # x*y=12
```

### Nested f-strings (3.12+)

```python
name = "world"
print(f"hello {f"{name!r}"}")    # works in 3.12+ — same quotes
```

---

## `pathlib.Path` — not `os.path`

```python
# OLD
import os
path = os.path.join("data", "raw", "users.csv")
if os.path.exists(path):
    with open(path) as f: data = f.read()
parent = os.path.dirname(path)

# NEW
from pathlib import Path
path = Path("data") / "raw" / "users.csv"
if path.exists():
    data = path.read_text()
parent = path.parent
```

Useful methods: `.exists()`, `.is_file()`, `.is_dir()`, `.read_text()`, `.write_text()`, `.read_bytes()`, `.suffix`, `.stem`, `.name`, `.parent`, `.parents`, `.glob()`, `.mkdir(parents=True, exist_ok=True)`, `.resolve()`.

---

## `dataclasses.field` features

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Config:
    items: list[str] = field(default_factory=list)
    secret: str = field(repr=False)                # excluded from __repr__
    cache: dict = field(default_factory=dict, compare=False)  # excluded from __eq__
```

Other useful flags: `kw_only=True` (force keyword-only), `slots=True` (memory savings), `match_args=False` (exclude from `match`).

---

## Iterators and itertools

### Itertools is your friend

```python
from itertools import chain, groupby, batched, pairwise, accumulate

list(chain([1, 2], [3, 4]))            # [1, 2, 3, 4]
list(batched("ABCDEFG", 3))            # [('A','B','C'), ('D','E','F'), ('G',)]  (3.12+)
list(pairwise([1, 2, 3, 4]))           # [(1,2), (2,3), (3,4)]
list(accumulate([1, 2, 3, 4]))         # [1, 3, 6, 10]
```

### Generators for one-shot data

```python
def lines_of(path: Path) -> Iterator[str]:
    with path.open() as f:
        yield from f

# Process a giant file without loading it all
for line in lines_of(Path("big.log")):
    if "ERROR" in line:
        print(line)
```

---

## `contextlib`

### `contextmanager` for ad-hoc context managers

```python
from contextlib import contextmanager

@contextmanager
def timing(label: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.3f}s")

with timing("expensive op"):
    do_thing()
```

### `suppress` to ignore specific exceptions cleanly

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    Path("cache.json").unlink()
```

Beats `try/except: pass`.

---

## `functools`

### `cache` / `lru_cache` for memoization

```python
from functools import cache

@cache
def fibonacci(n: int) -> int:
    if n < 2: return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

`cache` is `lru_cache(maxsize=None)` — unbounded. Use `lru_cache(maxsize=N)` if memory matters.

### `partial` for argument pre-binding

```python
from functools import partial

def send(method: str, url: str, *, headers: dict) -> Response: ...

post = partial(send, "POST")
post("/users", headers={"Authorization": "..."})
```

Cleaner than a closure for simple cases.

---

## Async (when it fits)

### `async def` + `await` for I/O-bound concurrency

```python
async def fetch_users(ids: list[int]) -> list[User]:
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_user(session, uid) for uid in ids]
        return await asyncio.gather(*tasks)
```

Async is for **I/O-bound** work: HTTP calls, database queries, file I/O. Not for CPU-bound work (use `multiprocessing` or threading). Don't async-ify everything; sync code is simpler when concurrency isn't needed.

### `TaskGroup` (3.11+) — structured concurrency

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch_users())
        tg.create_task(fetch_orders())
        tg.create_task(fetch_invoices())
    # All tasks completed (or one failed and the others were cancelled)
```

Replaces `asyncio.gather`, with proper cleanup semantics: if one task raises, the others are cancelled cleanly. Use `TaskGroup` over `gather` when failure semantics matter.

---

## Tooling

These aren't language features but they're part of "modern Python":

- **`uv`** — fast package manager (also handles Python versions). Replaces `pip`, `pip-tools`, `pyenv`, `poetry` for most use cases.
- **`ruff`** — fast linter + formatter. Replaces `black`, `flake8`, `isort`, `pylint`. One tool, one config.
- **`pyright`** or **`mypy`** — type checker. Pyright is faster and stricter; mypy is more established.
- **`pytest`** — testing. With `pytest-cov` for coverage, `pytest-asyncio` for async tests.
- **`pre-commit`** — runs ruff/pyright/pytest on every commit. Don't merge code that hasn't been checked.

The trio: **uv + ruff + pyright** is the modern default.

---

## Anti-features (don't use these)

- **`exec` / `eval`** on untrusted input. Almost never the right tool; usually a sign of bad design.
- **`from x import *`.** Ever.
- **`pickle` for cross-system or persisted data.** Use JSON, msgpack, or a schema'd format.
- **Mutable default arguments** (`def f(xs=[])`). Use `None` or `field(default_factory=list)`.
- **String formatting via `%`** (`"%s: %s" % (k, v)`). f-strings. Always f-strings.
- **`os.system` / `os.popen`** for shell commands. Use `subprocess.run(..., check=True)` with a list of args.
- **`time.sleep` in async code.** Use `await asyncio.sleep(...)`.

---

## A modern-Python snippet that uses many of these

```python
from dataclasses import dataclass, field
from decimal import Decimal
from enum import Enum, auto
from pathlib import Path
from typing import Self


class OrderStatus(Enum):
    PENDING = auto()
    PAID = auto()
    REFUNDED = auto()


@dataclass(frozen=True, slots=True)
class OrderItem:
    name: str
    price: Decimal
    qty: int

    @property
    def total(self) -> Decimal:
        return self.price * self.qty


@dataclass(frozen=True, slots=True)
class Order:
    id: int
    status: OrderStatus
    items: list[OrderItem] = field(default_factory=list)

    @property
    def total(self) -> Decimal:
        return sum((i.total for i in self.items), Decimal("0"))


class OrderBuilder:
    def __init__(self, id: int) -> None:
        self._id = id
        self._status = OrderStatus.PENDING
        self._items: list[OrderItem] = []

    def add(self, item: OrderItem) -> Self:
        self._items.append(item)
        return self

    def mark_paid(self) -> Self:
        self._status = OrderStatus.PAID
        return self

    def build(self) -> Order:
        return Order(id=self._id, status=self._status, items=self._items)


def describe(status: OrderStatus) -> str:
    match status:
        case OrderStatus.PENDING: return "awaiting payment"
        case OrderStatus.PAID: return "complete"
        case OrderStatus.REFUNDED: return "refunded"


order = (
    OrderBuilder(id=1)
        .add(OrderItem(name="Book", price=Decimal("15.00"), qty=2))
        .add(OrderItem(name="Pen", price=Decimal("2.50"), qty=3))
        .mark_paid()
        .build()
)

print(f"{order.id=}, status={describe(order.status)}, total={order.total}")
```

This is the baseline. If your Python doesn't look at least this modern, drag it forward.
