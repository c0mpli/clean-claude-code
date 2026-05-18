# Dates, Times, and Money

The two areas where Python code most often silently produces wrong answers. Both are about choosing the right type at the boundary and never letting the wrong one in.

---

## Money: `Decimal`, always

### Never use `float` for currency

Floating-point can't represent `0.1` exactly. Money math with `float` accumulates errors that show up in audits months later.

```python
# BAD — wrong in obvious cases, wronger in subtle ones
>>> 0.1 + 0.2
0.30000000000000004
>>> 1.10 * 3
3.3000000000000003

# GOOD
>>> from decimal import Decimal
>>> Decimal("0.1") + Decimal("0.2")
Decimal('0.3')
>>> Decimal("1.10") * 3
Decimal('3.30')
```

### Always construct `Decimal` from `str`, never from `float`

```python
# BAD — the float was already wrong; Decimal just preserves the wrongness
>>> Decimal(0.1)
Decimal('0.1000000000000000055511151231257827021181583404541015625')

# GOOD
>>> Decimal("0.1")
Decimal('0.1')
```

If your data source hands you a float (a JSON number, an unwrapped ORM column), **convert via `str` first** — or fix the source to use string-encoded decimals.

### Set precision and rounding mode explicitly

```python
from decimal import Decimal, ROUND_HALF_EVEN, getcontext

getcontext().prec = 28                  # 28 significant digits (default)

def quantize_money(amount: Decimal) -> Decimal:
    """Round to two decimal places, banker's rounding."""
    return amount.quantize(Decimal("0.01"), rounding=ROUND_HALF_EVEN)

quantize_money(Decimal("1.005"))        # → Decimal('1.00') (banker's: rounds to even)
quantize_money(Decimal("1.015"))        # → Decimal('1.02')
```

Default to `ROUND_HALF_EVEN` (banker's rounding) for money — it's what regulators expect and it doesn't bias accumulated totals. `ROUND_HALF_UP` is what humans usually mean by "round up at .5"; pick deliberately.

### Wrap money in a `Money` type — never bare `Decimal`

A bare `Decimal` doesn't know what currency it is. Mixing USD and EUR by accident is a silent bug.

```python
from decimal import Decimal
from typing import Self

class Currency(StrEnum):
    USD = "USD"
    EUR = "EUR"
    GBP = "GBP"

@dataclass(frozen=True, slots=True)
class Money:
    amount: Decimal
    currency: Currency

    def __post_init__(self) -> None:
        if not isinstance(self.amount, Decimal):
            raise TypeError(f"amount must be Decimal, got {type(self.amount).__name__}")

    def __add__(self, other: Self) -> Self:
        if self.currency is not other.currency:
            raise CurrencyMismatchError(self.currency, other.currency)
        return type(self)(amount=self.amount + other.amount, currency=self.currency)

    def __sub__(self, other: Self) -> Self:
        if self.currency is not other.currency:
            raise CurrencyMismatchError(self.currency, other.currency)
        return type(self)(amount=self.amount - other.amount, currency=self.currency)

    def __mul__(self, factor: Decimal | int) -> Self:
        return type(self)(amount=self.amount * Decimal(factor), currency=self.currency)


class CurrencyMismatchError(Exception):
    def __init__(self, a: Currency, b: Currency) -> None:
        self.a = a
        self.b = b
        super().__init__(f"cannot operate on {a.value} and {b.value} without conversion")
```

Now `Money(Decimal("10"), Currency.USD) + Money(Decimal("5"), Currency.EUR)` raises at the call site, not three layers deep when totals don't reconcile.

### Conversion is explicit and dated

Currency rates change. A conversion result is valid *as of* a timestamp. Don't pretend otherwise.

```python
@dataclass(frozen=True)
class ExchangeRate:
    base: Currency
    quote: Currency
    rate: Decimal
    as_of: datetime               # always tz-aware (see below)

def convert(amount: Money, to: Currency, rate: ExchangeRate) -> Money:
    if rate.base is not amount.currency or rate.quote is not to:
        raise ValueError(f"rate {rate.base.value}->{rate.quote.value} doesn't apply to {amount.currency.value}->{to.value}")
    return Money(amount=amount.amount * rate.rate, currency=to)
```

Never call a function like `to_usd(eur_amount)` that hides the rate.

### Tax, discounts, percentages

```python
# BAD — float drift
def with_tax(price: float, rate: float = 0.07) -> float:
    return price * (1 + rate)

# GOOD
def with_tax(price: Decimal, rate: Decimal = Decimal("0.07")) -> Decimal:
    return quantize_money(price * (Decimal("1") + rate))
```

Quantize the final result, not intermediate values — rounding mid-pipeline biases totals.

---

## Datetime: always timezone-aware

### Never `datetime.now()` without a timezone

Naive datetimes are the source of an entire genre of bugs. Two servers in two timezones disagree about "today". DST transitions silently swallow or duplicate an hour.

```python
# BAD
from datetime import datetime
created_at = datetime.now()                              # naive, ambiguous
created_at = datetime.utcnow()                           # naive AND deprecated in 3.12

# GOOD
from datetime import datetime, UTC
created_at = datetime.now(UTC)                           # aware
```

`datetime.utcnow()` is **deprecated** as of Python 3.12. Use `datetime.now(UTC)`.

### Use `zoneinfo`, not `pytz`

```python
from datetime import datetime
from zoneinfo import ZoneInfo

new_york = ZoneInfo("America/New_York")
now_ny = datetime.now(new_york)

# Localize an existing aware datetime
utc_dt = datetime(2026, 6, 1, 12, 0, tzinfo=UTC)
local_dt = utc_dt.astimezone(new_york)                   # converts to NY time
```

`zoneinfo` is stdlib (3.9+). `pytz` is legacy and has a quirky API (`.localize()` vs `.replace(tzinfo=...)`). New code uses `zoneinfo`.

### Storage in UTC, display in local

The rule:
- **Persisted timestamps** (database, logs, queue messages, API responses) → **UTC**.
- **Display to users** → convert to their local timezone at the last moment.

```python
@dataclass(frozen=True)
class Event:
    id: str
    occurred_at: datetime                                # stored as UTC

    def __post_init__(self) -> None:
        if self.occurred_at.tzinfo is None:
            raise ValueError("occurred_at must be timezone-aware")
        if self.occurred_at.utcoffset() != timedelta(0):
            raise ValueError("occurred_at must be in UTC")


def format_for_user(event: Event, tz: ZoneInfo) -> str:
    local = event.occurred_at.astimezone(tz)
    return local.strftime("%Y-%m-%d %H:%M %Z")
```

The `__post_init__` is the boundary. Once an `Event` exists, the timezone is guaranteed.

### Parsing — `fromisoformat`, not `strptime` of arbitrary strings

```python
# BAD — fragile, locale-dependent, slow
datetime.strptime("2026-05-18T12:34:56", "%Y-%m-%dT%H:%M:%S")

# GOOD — ISO 8601 in/out
datetime.fromisoformat("2026-05-18T12:34:56+00:00")      # aware
datetime.fromisoformat("2026-05-18T12:34:56Z")           # aware (3.11+)
```

Boundary layer (HTTP, JSON): use Pydantic, which validates and parses to aware `datetime` for you.

```python
from pydantic import BaseModel, AwareDatetime

class CreateEvent(BaseModel):
    occurred_at: AwareDatetime                           # rejects naive datetimes
```

### Never parse dates with string splits

```python
# BAD
year, month, day = map(int, date_str.split("-"))         # breaks on "2026-05-18T12:00:00"
month, day, year = date_str.split("/")                   # ambiguous: 05/06 — May 6 or June 5?

# GOOD
dt = datetime.fromisoformat(date_str)
```

If you receive a date in some weird format (`MM/DD/YYYY`), use `strptime` once at the boundary with the exact format. Never `split` your way through it.

### Arithmetic — `timedelta` always

```python
# BAD
expiry = now + 24 * 60 * 60                              # int seconds; lossy

# GOOD
from datetime import timedelta
expiry = now + timedelta(hours=24)
expiry = now + timedelta(days=7, hours=3)
```

For month / year arithmetic (which `timedelta` can't do because months aren't a fixed length), use `dateutil.relativedelta`:

```python
from dateutil.relativedelta import relativedelta

next_month   = now + relativedelta(months=1)
last_year    = now - relativedelta(years=1)
end_of_month = now + relativedelta(day=31)               # clamps to actual last day
```

### Comparisons — never mix naive and aware

```python
naive   = datetime(2026, 1, 1)
aware   = datetime(2026, 1, 1, tzinfo=UTC)
naive < aware                                            # TypeError
```

Python refuses the comparison. Good. Make every datetime aware at the boundary, then mixing is impossible.

### Durations and intervals

Use `timedelta` for durations. Use the `interval` types from `pendulum` or `arrow` only if you need richer interval algebra (you almost never do).

```python
from datetime import timedelta

session_timeout: Final = timedelta(minutes=30)
retry_delay:     Final = timedelta(seconds=5)
cache_ttl:       Final = timedelta(hours=1)

if now - session.created_at > session_timeout:
    raise SessionExpiredError(session.id)
```

Never store durations as bare ints (`session_timeout = 1800`). The reader has to remember the unit. `timedelta(minutes=30)` is self-documenting.

---

## Sleep, intervals, scheduling

```python
# BAD
import time
time.sleep(60 * 5)                                       # 5 minutes? 5 hours? Hard to scan.

# GOOD
import time
time.sleep(timedelta(minutes=5).total_seconds())

# BETTER — wrap it
def sleep_for(d: timedelta) -> None:
    time.sleep(d.total_seconds())

sleep_for(timedelta(minutes=5))
```

In async code: `await asyncio.sleep(timedelta(...).total_seconds())`. Never `time.sleep` in async — it blocks the event loop.

---

## Anti-patterns

| Symptom | Fix |
|---|---|
| `float` for money | `Decimal` |
| `Decimal(0.1)` (from float) | `Decimal("0.1")` (from str) |
| Bare `Decimal` for currency | `Money(amount, currency)` |
| `datetime.now()` (no tz) | `datetime.now(UTC)` |
| `datetime.utcnow()` | Deprecated — use `datetime.now(UTC)` |
| `import pytz` | `from zoneinfo import ZoneInfo` |
| Storing local time | Store UTC; convert for display |
| `datetime.strptime("...", "%Y-%m-%d...")` for ISO | `datetime.fromisoformat(...)` |
| Splitting strings to parse dates | `fromisoformat` or `strptime` at boundary |
| `expires_at = now + 86400` | `expires_at = now + timedelta(days=1)` |
| `timedelta(months=1)` (doesn't exist) | `relativedelta(months=1)` |
| `if naive < aware:` | Make both aware at the boundary |
| `time.sleep(1800)` | `time.sleep(timedelta(minutes=30).total_seconds())` |
| `time.sleep(...)` in async code | `await asyncio.sleep(...)` |
| Tax math with `Decimal * 0.07` | `Decimal * Decimal("0.07")` |
| Quantizing intermediate values | Quantize only the final result |

---

## The one-line rules

- **Money:** `Decimal` from `str`, wrapped in `Money(amount, currency)`, quantize only at the end.
- **Time:** timezone-aware always, UTC for storage, `zoneinfo` for zones, ISO 8601 at boundaries, `timedelta` for durations.
- **Both:** validate at the constructor (`__post_init__`), never let a `float`-money or naive-`datetime` exist as a live value in your program.
