# Error Handling

How to model failure, when to use exceptions vs `Result` types, and how to make errors useful instead of frustrating.

## The two kinds of failure

Treat these differently:

1. **Exceptional**: something that shouldn't normally happen — corrupt state, programmer errors, network outage. Use exceptions.
2. **Expected**: parts of normal control flow — parsing user input, looking up a missing key, validating a form. Use return types: `T | None`, `Result[T, E]`, or domain-specific success/failure types.

When `try/except` is wrapping the *normal* happy path of a function, you've miscategorized the failure. Return a value instead.

---

## Custom exceptions with context

### Rule: every domain exception has structured fields

Never raise `ValueError("something went wrong")`. Define a domain exception class that carries the relevant data as attributes. Callers can branch on the class and inspect fields without parsing `str(e)`.

```python
class NotEnoughVacationError(Exception):
    def __init__(self, requested: int, remaining: int) -> None:
        self.requested = requested
        self.remaining = remaining
        super().__init__(
            f"Requested {requested} days, only {remaining} remaining"
        )

# Caller can branch on the exception type AND read the fields
try:
    employee.payout_holiday()
except NotEnoughVacationError as e:
    log.warning("payout refused: requested=%d remaining=%d", e.requested, e.remaining)
    notify_user_low_balance(employee, e.remaining)
```

### Domain exception hierarchies

If a module raises several related exceptions, give them a common base so callers can choose granularity:

```python
class InvoiceError(Exception): ...
class InvoiceNotFoundError(InvoiceError):
    def __init__(self, invoice_id: str) -> None:
        self.invoice_id = invoice_id
        super().__init__(f"Invoice {invoice_id} not found")
class InvoiceAlreadyPaidError(InvoiceError):
    def __init__(self, invoice_id: str, paid_at: datetime) -> None:
        self.invoice_id = invoice_id
        self.paid_at = paid_at
        super().__init__(f"Invoice {invoice_id} already paid at {paid_at}")
```

Callers can `except InvoiceError` to handle anything from this module, or `except InvoiceNotFoundError` for one specific case.

---

## Where to validate

### Fail fast at construction time

If an object has invariants ("`age` must be 13–120", "`output_dir` must be writable"), enforce them in `__post_init__` (dataclass) or `model_validator` (Pydantic). Don't let a half-broken object propagate through five function calls before exploding.

```python
@dataclass(frozen=True)
class TrainingConfig:
    data_path: Path
    test_size: float

    def __post_init__(self) -> None:
        if not self.data_path.exists():
            raise FileNotFoundError(f"Dataset not found: {self.data_path}")
        if not 0 < self.test_size < 1:
            raise ValueError(f"test_size must be in (0, 1), got {self.test_size}")
```

### Validate at the boundary

Untrusted input (HTTP body, config file, CLI arg) gets parsed by a Pydantic model at the edge. Internal code is then free to trust its types.

```python
@app.post("/users")
async def create_user(req: CreateUserRequest) -> User:
    # req is already validated. No try/except for "is email a string?"
    return await user_service.create(req.email, req.password)
```

### Don't double-validate

Once a value has been through Pydantic, downstream code trusts it. If you find yourself re-checking `if user.email is None` inside the service, the boundary types are wrong.

### Every public method validates its preconditions

The "system boundary" rule extends to **every public method**: validate the inputs you don't already trust by type. Private helpers (prefixed `_`) trust their callers — the public surface protects them.

```python
@dataclass(frozen=True)
class VehicleInfo:
    brand: str
    electric: bool
    catalogue_price: Decimal

    def compute_tax(self, exemption: Decimal = Decimal("0")) -> Decimal:
        if exemption < 0:
            raise ValueError(f"exemption must be non-negative, got {exemption}")
        return self._tax_rate() * max(self.catalogue_price - exemption, Decimal("0"))

    def can_lease(self, year_income: Decimal) -> bool:
        if year_income < 0:
            raise ValueError(f"year_income must be non-negative, got {year_income}")
        return self.catalogue_price <= year_income * Decimal("0.7")

    def _tax_rate(self) -> Decimal:                          # private — trusts the caller
        return Decimal("0.02") if self.electric else Decimal("0.05")
```

The type hint says `Decimal`. The precondition check says *which* `Decimal`s are valid. Types narrow the space; preconditions narrow it further.

Two practical rules:
- The check is **at the top of the method**, not deep inside. Fail at the entry, not after three operations.
- The error message **names the parameter and shows the bad value**. `"exemption must be non-negative, got -100"` is debuggable; `"invalid input"` is not.

---

## Result types — for expected failure

When failure is a normal part of control flow, return it as data. Either:

- **`T | None`** — simplest, when "didn't find it" / "wasn't valid" is the only failure mode.
- **`Result[T, E]`** (from the `returns` library, or hand-rolled) — when there are multiple distinct failure kinds.
- **A dedicated sum type** — `Success(value) | Failure(reason)` as your own dataclasses.

### `T | None` for "find or miss"

```python
def find_user(user_id: int) -> User | None:
    return db.get(user_id)

# Caller handles missing case
if (user := find_user(uid)) is None:
    return Response(404)
```

Use `T | None`, not `Optional[T]`. Same meaning, less noise.

### `Result[T, E]` for chainable failure

```python
from returns.result import Failure, Result, Success

def parse_number(s: str) -> Result[int, ParseError]:
    try:
        return Success(int(s))
    except ValueError:
        return Failure(ParseError(input=s))

def add_ten(n: int) -> Result[int, ParseError]:
    return Success(n + 10)

# Chain — short-circuits on Failure automatically
result = parse_number(user_input).bind(add_ten)
match result:
    case Success(value):
        print(value)
    case Failure(err):
        print(f"Parse failed: {err.input!r}")
```

`Result` is great when you have a pipeline of operations any of which can fail, and you want to chain without nested try/except.

---

## Never do this

### `except:` or `except Exception:` at random

```python
# NEVER
try:
    do_thing()
except:                       # catches KeyboardInterrupt, SystemExit, everything
    pass
except Exception:             # only marginally less bad
    pass
```

Bare `except` swallows `KeyboardInterrupt` and `SystemExit`. `except Exception` is almost as bad — you've decided that "whatever went wrong, ignore it" is fine. It's never fine.

```python
# OK
try:
    do_thing()
except SpecificError as e:
    log.warning("expected failure: %s", e)
    return fallback
```

Catch the specific exception you know how to handle. Let the rest propagate.

### Lossy re-raise

```python
# BAD — original traceback lost
try:
    parse(text)
except ValueError:
    raise MyError("parse failed")

# GOOD — chain via `from`
try:
    parse(text)
except ValueError as e:
    raise MyError(f"parse failed: {text!r}") from e
```

Using `from e` preserves the original traceback for debugging.

### Returning sentinel values to indicate failure

```python
# BAD
def find_user(uid: int) -> User:
    user = db.get(uid)
    if user is None:
        return None              # but the return type says User!
    return user

# Worse
def parse_int(s: str) -> int:
    try:
        return int(s)
    except ValueError:
        return -1                # is -1 a valid result? unclear forever after.

# GOOD
def find_user(uid: int) -> User | None: ...
def parse_int(s: str) -> int | None: ...
```

### Catching just to log and re-raise

```python
# BAD — adds nothing
try:
    do_thing()
except Exception as e:
    log.error(e)
    raise

# Just let it propagate. Configure logging to capture unhandled exceptions
# at the top of the call stack, not inside every function.
```

---

## Logging

- **Log at the boundary**, not at the throw site. Whoever has enough context to handle the error logs it.
- **Use structured logging** — `log.info("created user", extra={"user_id": user.id})` — not f-strings into the message text. Makes logs queryable.
- **Don't log and raise.** Pick one. If you can handle it, log and recover. If you can't, raise and let someone higher up log + handle.
- **Never log secrets.** This is what `SecretStr` is for.

---

## Quick decision tree

When something can go wrong, ask:

1. **Is this a programmer error / corrupt state?** → Raise. (`AssertionError`, `TypeError`, `RuntimeError`.)
2. **Is this an environmental failure (disk full, network down)?** → Raise a domain exception (`StorageUnavailableError`) with context. Let the caller decide whether to retry or surface to the user.
3. **Is this a user-input or external-data failure?** → Validate at the boundary (Pydantic). The validation failure is itself a value the boundary returns (a 400 response, an error message).
4. **Is this a normal "didn't find it" / "wasn't valid" case in the middle of business logic?** → Return `T | None` or `Result[T, E]`.

The wrong move 90% of the time is "raise a generic `ValueError` mid-pipeline". The right move is one of the four above.
