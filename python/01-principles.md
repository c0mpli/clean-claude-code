# The 12 Principles

Each principle is a prescriptive rule with a one-paragraph rationale and a minimal example. Deep dives are in the other files.

---

## 1. One function, one thing

**Rule:** If you can't describe a function in one sentence without using the word "and", split it. Extract shared logic into a private helper so the split doesn't create duplication.

```python
# BAD — boolean flag means two responsibilities
def take_holiday(self, payout: bool) -> None:
    if payout:
        if self.vacation_days < 5: raise ValueError(...)
        self.vacation_days -= 5
    else:
        if self.vacation_days < 1: raise ValueError(...)
        self.vacation_days -= 1

# GOOD — two functions, one private helper for the shared bit
def payout_holiday(self) -> None:
    self._withdraw_holiday(5)

def take_single_holiday(self) -> None:
    self._withdraw_holiday(1)

def _withdraw_holiday(self, days: int) -> None:
    if self.vacation_days < days:
        raise NotEnoughVacationError(days, self.vacation_days)
    self.vacation_days -= days
```

Splitting added a function but **prevented** duplication. Call sites read like English. See `02-functions.md`.

---

## 2. Push variation into types, not conditionals

**Rule:** If you find yourself writing `if isinstance(x, A): ... elif isinstance(x, B): ...`, or `if kind == "foo": ... elif kind == "bar": ...`, the variation belongs in the type system — use polymorphism, `Protocol`, a dispatch dict, or `match`.

```python
# BAD
def pay(employee):
    if isinstance(employee, Salaried):
        print(f"${employee.monthly_salary}")
    elif isinstance(employee, Hourly):
        print(f"${employee.hourly_rate} × {employee.hours}")

# GOOD — each type owns its behavior
class Employee(ABC):
    @abstractmethod
    def pay(self) -> None: ...

@dataclass
class Hourly(Employee):
    hourly_rate: float
    hours: int
    def pay(self) -> None:
        print(f"${self.hourly_rate} × {self.hours}")
```

---

## 3. Type-hint every signature

**Rule:** Every parameter and return type is annotated. Use modern syntax: `str | None`, `list[int]`, `dict[str, T]`. Use `Protocol` for structural typing. Use PEP 695 `[T]` for generics. Type hints are documentation that the type checker enforces.

```python
def find_employees(self, role: Role) -> list[Employee]:
    return [e for e in self.employees if e.role is role]
```

---

## 4. No boolean flag parameters

**Rule:** A `flag: bool` parameter that switches behavior is two functions in a trench coat. Split them.

```python
# BAD
def render(html: str, minify: bool) -> str: ...
render(html, minify=True)   # what does True mean at the call site?

# GOOD
def render(html: str) -> str: ...
def render_minified(html: str) -> str: ...
```

Exception: a boolean parameter that toggles a single line of *configuration* (not behavior) is fine — `Path(...).mkdir(exist_ok=True)`.

---

## 5. No magic strings

**Rule:** String literals that represent a fixed set of values are `Enum`s.

```python
# BAD
if order.status == "paid": ...
employee.role == "vice_president"

# GOOD
class OrderStatus(Enum):
    PAID = auto()
    PENDING = auto()
    REFUNDED = auto()

if order.status is OrderStatus.PAID: ...
```

Use `is` (identity), not `==`, when comparing enums — it's faster and signals "this is a singleton, not a value comparison".

---

## 6. No wildcard imports

**Rule:** Never `from x import *`. Either `import x` (and call as `x.thing`) or `from x import thing` (named). Wildcard imports pollute the namespace, defeat linters, and obscure where names come from.

---

## 7. Validate at boundaries, trust internally

**Rule:** Use Pydantic at the system edge (HTTP request bodies, config files, external APIs). Use plain dataclasses inside the application. Validation happens once, at the boundary; internal code trusts the types.

```python
# Boundary
class CreateUserRequest(BaseModel):
    email: EmailStr
    password: SecretStr
    age: int = Field(ge=13, le=120)

# Internal (after validation)
@dataclass(frozen=True)
class User:
    email: str
    age: int
```

---

## 8. Frozen dataclasses for value objects

**Rule:** Default to `@dataclass(frozen=True)` for domain objects. Mutable dataclasses are opt-in. Frozen objects are hashable (usable as dict keys / in sets), thread-safe, and prevent the "who mutated this?" debugging session.

```python
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str
```

---

## 9. Custom exceptions carry context

**Rule:** Never raise a bare `ValueError("something went wrong")`. Define a domain exception class with the relevant data as attributes so callers can branch on the exception, not parse `str(e)`.

```python
class NotEnoughVacationError(Exception):
    def __init__(self, requested: int, remaining: int) -> None:
        self.requested = requested
        self.remaining = remaining
        super().__init__(
            f"Requested {requested} days, only {remaining} remaining"
        )
```

---

## 10. Names carry documentation

**Rule:** Functions start with a verb. Variables are nouns. Use full words, not abbreviations. **Never** put type info in a name — the type hint already says that.

```python
# BAD
def total(items, d): ...
def calculate_integer_total_price(items: list[int]) -> int: ...
c, cust, vc  # cryptic vars

# GOOD
def calculate_total_price(items: list[int], discount: int) -> int: ...
customer, vacation_count
```

A function name should describe what the function *does*, not how it does it.

---

## 11. List comprehensions over `for ... append`

**Rule:** When you're transforming or filtering an iterable into a list/dict/set, use a comprehension. Reserve `for` loops for actions with side effects.

```python
# BAD
result = []
for order in orders:
    if order.status is OrderStatus.PAID:
        result.append(order.total)

# GOOD
paid_totals = [o.total for o in orders if o.status is OrderStatus.PAID]
```

---

## 12. Dict lookup over list scan

**Rule:** When you'll look up an item by key more than once, build a dict. O(1) beats O(n). Use tuples as composite keys when the lookup needs multiple fields.

```python
# BAD
def find_vehicle(brand: str, model: str) -> VehicleInfo | None:
    for v in self.vehicles:
        if v.brand == brand and v.model == model:
            return v
    return None

# GOOD
self.vehicles: dict[tuple[str, str], VehicleInfo] = {}
def find_vehicle(self, brand: str, model: str) -> VehicleInfo | None:
    return self.vehicles.get((brand, model))
```

---

## Bonus tier (strongly recommended, occasionally context-dependent)

- **Group related parameters into a config dataclass.** When a function takes 4+ related args, group them.
- **`@property` for cheap computed values.** `order.total`, not `order.calculate_total()`.
- **`__post_init__` for invariant checks.** Fail fast at construction time.
- **`@staticmethod` (or module-level function) when `self` is unused.** Don't fake instance-ness.
- **Use `Self` as the return type for fluent builders.** Enables chaining + works with subclasses.
- **Replace nested `if`s with early returns.** Reduce indentation depth.
- **Use the walrus operator `:=` for assign-and-test.** Makes "get-or-fail" patterns concise.
- **Prefer `match` for shape-based dispatch.** When `isinstance` is unavoidable, `match` makes it readable.

These principles aren't aspirations. They're the floor. Code that violates them is incorrect and needs to be rewritten.
