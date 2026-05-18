# Data & Types

How to model data, how to use the type system, and when each tool fits.

## The hierarchy of choices

When you need a new type, pick the *least* powerful tool that fits:

1. **Built-in (`int`, `str`, `Decimal`, `Path`)** — use directly when it's just a value.
2. **`Enum`** — fixed set of named values.
3. **`@dataclass(frozen=True)`** — immutable structured value with multiple fields. **The default.**
4. **`@dataclass`** — same, but mutable. Use only when you actually need to mutate.
5. **`pydantic.BaseModel`** — at system boundaries (HTTP, config files, external APIs). Validates and parses.
6. **`class` with methods** — when behavior is non-trivial and tied to the data.
7. **`Protocol`** — structural interface. Several unrelated classes implement the same shape.
8. **`ABC` + `abstractmethod`** — nominal interface. You control the hierarchy and want runtime enforcement.

When in doubt: frozen dataclass.

---

## Dataclasses

### Default to frozen

```python
@dataclass(frozen=True)
class Customer:
    id: int
    name: str
    tier: Tier
```

Frozen means hashable (usable as dict keys / in sets), thread-safe, and eliminates an entire class of "who mutated this?" bugs. Use mutable dataclasses only when the object's identity genuinely changes over time (e.g., stateful business entities, accumulators).

### `__post_init__` for invariants

Validate invariants in `__post_init__`. Fail at construction time, not deep inside a function.

```python
@dataclass(frozen=True)
class TrainingConfig:
    data_path: Path
    output_dir: Path
    test_size: float = 0.2
    random_state: int = 42

    def __post_init__(self) -> None:
        if not self.data_path.exists():
            raise FileNotFoundError(f"Dataset not found: {self.data_path}")
        if self.data_path.suffix != ".csv":
            raise ValueError("data_path must be a CSV file")
        if not 0 < self.test_size < 1:
            raise ValueError("test_size must be in (0, 1)")
```

### `field(default_factory=...)` for mutable defaults

Never write `def f(x=[])`. In dataclasses, use `field(default_factory=list)`:

```python
@dataclass(frozen=True)
class Order:
    customer_id: int
    items: list[OrderItem] = field(default_factory=list)
```

### `@property` for cheap computed values

If a value is derived from other fields and is cheap to compute, expose it as a property — not a method.

```python
@dataclass(frozen=True)
class Order:
    items: list[OrderItem]

    @property
    def total(self) -> Decimal:
        return sum((i.price * i.quantity for i in self.items), Decimal("0"))
```

Callers write `order.total`, not `order.calculate_total()`. Make sure properties stay cheap — surprise O(n) work behind attribute access is a footgun.

### Slots for tight memory

For hot-path objects with many instances, add `slots=True`:

```python
@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float
```

Don't sprinkle this everywhere; reach for it when profiling tells you to.

---

## Enums

### Replace fixed string sets with Enum

```python
class OrderStatus(Enum):
    PENDING = auto()
    PAID = auto()
    REFUNDED = auto()

class Tier(Enum):
    STANDARD = auto()
    PREMIUM = auto()

# Use `is` for comparison (faster, signals singleton)
if order.status is OrderStatus.PAID: ...
if customer.tier is Tier.PREMIUM: ...
```

### `StrEnum` when you need string serialization

When the enum's value will be serialized as a string (JSON APIs, database columns), use `StrEnum` (Python 3.11+) so members are also strings:

```python
class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"
    REFUNDED = "refunded"

# OrderStatus.PAID == "paid"  # True — useful for JSON
# But internally still: if order.status is OrderStatus.PAID
```

### `IntEnum` for legacy integer codes

Same idea, but for ints. Don't reach for it unless an external system requires integer codes.

---

## Pydantic — at the boundary, only

### Rule: validate at the edge, trust internally

Pydantic belongs at the **edge** of your system — anywhere data enters from outside (HTTP, config files, external APIs, CLI args). Inside the application, use plain dataclasses.

```python
# Boundary — Pydantic
class CreateUserRequest(BaseModel):
    email: EmailStr
    password: SecretStr
    age: int = Field(ge=13, le=120)
    role: Role = Role.MEMBER

# Internal — plain frozen dataclass
@dataclass(frozen=True)
class User:
    email: str
    age: int
    role: Role
```

Why: Pydantic validation is expensive. You want it run *once*, at the boundary. Internal code can trust the dataclass.

### Useful Pydantic field types

- `EmailStr` — validates email format.
- `SecretStr` — wraps a secret so it doesn't accidentally appear in logs/repr.
- `Field(ge=, le=, gt=, lt=)` — numeric bounds.
- `Field(min_length=, max_length=)` — string bounds.
- `Field(frozen=True)` — field-level immutability.
- `Field(default_factory=...)` — same idea as dataclass `field`.

### `model_validate` and `model_dump`

```python
try:
    user = User.model_validate(request_json)
except ValidationError as e:
    for error in e.errors():
        log.warning(error)

serialized = user.model_dump()                  # → dict
serialized_json = user.model_dump_json()        # → str
```

Never use the deprecated `parse_obj` / `dict()` / `json()` — those are Pydantic v1.

---

## Protocol — structural typing

Use `Protocol` when several unrelated classes implement the same shape, and you want to type-check against the shape without forcing inheritance.

```python
from typing import Protocol

class Writable(Protocol):
    def write(self, data: bytes) -> int: ...

def save(stream: Writable, data: bytes) -> int:
    return stream.write(data)

# Anything with .write(bytes) -> int satisfies this:
save(open("f", "wb"), b"hi")
save(io.BytesIO(), b"hi")
save(MyCustomStream(), b"hi")
```

When to prefer Protocol over ABC:

- You don't control the classes (e.g., third-party libs).
- You want duck-typing with static checks.
- You don't need `isinstance(x, Protocol)` checks at runtime.

When to prefer ABC over Protocol:

- You want to share concrete method implementations (not just signatures).
- You want runtime enforcement (`isinstance` checks).
- The hierarchy is yours and is naturally tree-shaped.

### `@runtime_checkable` if you really need isinstance

```python
@runtime_checkable
class Writable(Protocol):
    def write(self, data: bytes) -> int: ...

isinstance(obj, Writable)   # works, but only checks method names, not signatures
```

Use sparingly. Runtime structural checks are weaker than they look.

---

## Generics (PEP 695)

Use the modern syntax. `TypeVar` is legacy.

### Generic functions

```python
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

# Constrained
def add_to_each[N: (int, float, Decimal)](n: N, items: list[N]) -> list[N]:
    return [x + n for x in items]
```

### Generic classes

```python
class Repository[T](ABC):
    @abstractmethod
    def get(self, id: int) -> T: ...
    @abstractmethod
    def list_all(self) -> list[T]: ...

class PostRepository(Repository[Post]):
    def get(self, id: int) -> Post: ...
    def list_all(self) -> list[Post]: ...
```

### Bounded vs constrained

- **Bounded** (`[T: SomeClass]`): T must be SomeClass or a subclass.
- **Constrained** (`[T: (A, B, C)]`): T must be exactly A, B, or C.

```python
# Bounded — works on any Animal, returns the same subtype
def feed[A: Animal](animal: A) -> A: ...

# Constrained — only int, float, or Decimal; types stay consistent
def add[N: (int, float, Decimal)](a: N, b: N) -> N: ...
```

---

## Type aliases

For complex types you reuse, define an alias with the `type` statement (Python 3.12+):

```python
type Vector = list[float]
type Matrix = list[Vector]
type ErrorOr[T] = T | Exception

def dot(a: Vector, b: Vector) -> float: ...
```

For simpler one-off types, inline them. Aliases pay off when the same type appears in 3+ signatures.

---

## `Self` for fluent return types

When a method returns its own instance (typically in builders), use `Self`:

```python
from typing import Self

class QueryBuilder:
    def where(self, condition: str) -> Self:
        self._conditions.append(condition)
        return self

    def limit(self, n: int) -> Self:
        self._limit = n
        return self

    def build(self) -> Query:
        return Query(self._conditions, self._limit)
```

`Self` works correctly with subclasses; `-> QueryBuilder` would break subclass type inference.

---

## `Literal` for enum-like ints/strs you can't refactor

When you can't or shouldn't make something an `Enum` (e.g., it's a public API that takes string codes), use `Literal`:

```python
def set_log_level(level: Literal["debug", "info", "warning", "error"]) -> None: ...
```

The type checker enforces the choice; users still pass plain strings.

---

## `Final` for module-level constants

```python
from typing import Final

MAX_RETRIES: Final = 3
DEFAULT_TIMEOUT: Final[float] = 30.0
```

The checker flags rebinding `MAX_RETRIES`. Documents intent.

---

## What to never do

- **Never** use `Any` to silence the type checker. If you reach for `Any`, you're admitting the type is wrong. Find the right type.
- **Never** use `# type: ignore` without a reason in a comment.
- **Never** mix Pydantic models and dataclasses in the same role within one layer. Pick one per layer (boundary vs internal).
- **Never** mutate a `frozen=True` dataclass with `object.__setattr__`. If you need to, the dataclass shouldn't be frozen.
- **Never** define an `Enum` where members aren't an exhaustive list of valid values. Adding a member shouldn't be a "let me also fix five `match` statements".
