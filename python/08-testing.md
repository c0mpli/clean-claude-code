# Testing

Tests are not optional. Untested code is broken code that hasn't been observed failing yet. Every rule in this guide exists to make code easier to test; this file is how you actually do it.

---

## The framework

**Use `pytest`.** Not `unittest`. Pytest is the standard.

- Plain functions instead of test classes.
- `assert` instead of `assertEqual` / `assertTrue` / `assertRaises`.
- `pytest.raises` instead of `assertRaises`.
- `@pytest.mark.parametrize` instead of eight near-identical tests.
- `@pytest.fixture` instead of `setUp` / `tearDown`.

```python
# BAD — unittest
class TestVehicleInfo(unittest.TestCase):
    def test_compute_tax_non_electric(self):
        v = VehicleInfo("BMW", electric=False, catalogue_price=10_000)
        self.assertEqual(v.compute_tax(), 500)

# GOOD — pytest
def test_compute_tax_non_electric():
    v = VehicleInfo(brand="BMW", electric=False, catalogue_price=Decimal("10000"))
    assert v.compute_tax() == Decimal("500")
```

---

## Test layout

Mirror the source tree. One test file per module.

```
src/
  billing/
    invoice.py
    tax.py
tests/
  billing/
    test_invoice.py
    test_tax.py
```

Run from the repo root: `pytest`. Configure once in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --cov=src --cov-report=term-missing"
```

---

## Test naming

**Pattern:** `test_<unit>_<scenario>[_<expectation>]`.

```python
def test_compute_tax_non_electric():                          # default scenario
def test_compute_tax_electric():                              # variant
def test_compute_tax_exemption_reduces_total():               # what it does
def test_compute_tax_negative_exemption_raises():             # error case
def test_compute_tax_exemption_above_price_returns_zero():    # edge
def test_can_lease_returns_true_when_price_under_threshold():
def test_can_lease_returns_false_when_price_over_threshold():
def test_can_lease_negative_income_raises():
```

A failing test's name should tell you what's broken without opening the file. `test_tax_1`, `test_works`, `test_edge_case` tell you nothing — rename them.

---

## One behavior per test

Each test asserts one behavior. Three behaviors → three tests.

```python
# BAD — one test, three behaviors, one failure hides the others
def test_invoice():
    inv = Invoice(items=[LineItem("a", 1, Decimal("10"))])
    assert inv.subtotal == Decimal("10")
    assert inv.tax == Decimal("0.50")
    assert inv.total == Decimal("10.50")

# GOOD — split
def test_invoice_subtotal_sums_line_items(): ...
def test_invoice_tax_is_five_percent_of_subtotal(): ...
def test_invoice_total_includes_tax(): ...
```

Exception: asserting on multiple fields of the *same* result object (one dataclass with five fields) is one behavior.

---

## AAA — arrange / act / assert

Every test has three sections, separated by blank lines. No labelling comments needed — the blank lines are the structure.

```python
def test_payout_holiday_reduces_vacation_days():
    employee = Employee(name="Arjan", vacation_days=10)

    employee.payout_holiday()

    assert employee.vacation_days == 5
```

If a test doesn't fit this shape, it's probably testing more than one thing.

---

## `pytest.raises` for error cases

```python
def test_payout_holiday_with_insufficient_days_raises():
    employee = Employee(name="Arjan", vacation_days=3)

    with pytest.raises(NotEnoughVacationError) as exc_info:
        employee.payout_holiday()

    assert exc_info.value.requested == 5
    assert exc_info.value.remaining == 3
```

Assert on the exception's **structured fields** — not `str(e)`. This is why custom exceptions carry attributes (rule #9). Tests that grep `"not enough days" in str(e)` rot the moment someone improves the wording.

Use `match=` for a quick regex check only when there are no structured fields:

```python
with pytest.raises(ValueError, match=r"must be non-negative"):
    compute_tax(exemption=Decimal("-1"))
```

---

## `parametrize` over copy-pasted tests

When the same logic runs against multiple inputs, parametrize.

```python
# BAD — five tests, identical body
def test_compute_tax_non_electric_low():
    assert VehicleInfo("X", False, Decimal("10000")).compute_tax() == Decimal("500")
def test_compute_tax_non_electric_high():
    assert VehicleInfo("X", False, Decimal("50000")).compute_tax() == Decimal("2500")
# ...

# GOOD — one test, table of cases
@pytest.mark.parametrize("electric,price,expected", [
    (False, Decimal("10000"), Decimal("500")),
    (False, Decimal("50000"), Decimal("2500")),
    (True,  Decimal("10000"), Decimal("200")),
    (True,  Decimal("50000"), Decimal("1000")),
])
def test_compute_tax(electric, price, expected):
    v = VehicleInfo(brand="X", electric=electric, catalogue_price=price)
    assert v.compute_tax() == expected
```

Use `pytest.param(..., id="...")` for readable test IDs in the output:

```python
@pytest.mark.parametrize("status,can_refund", [
    pytest.param(OrderStatus.PAID,     True,  id="paid-can-refund"),
    pytest.param(OrderStatus.PENDING,  False, id="pending-cannot-refund"),
    pytest.param(OrderStatus.REFUNDED, False, id="already-refunded"),
])
def test_can_refund(status, can_refund):
    assert Order(id=1, status=status).can_refund() is can_refund
```

`-v` then prints `test_can_refund[paid-can-refund] PASSED`. Much better than `test_can_refund[OrderStatus.PAID-True]`.

---

## Fixtures — for shared setup

A fixture is a function that produces a value for a test. Use them for construction reused across tests.

```python
@pytest.fixture
def employee() -> Employee:
    return Employee(name="Arjan", vacation_days=10)

def test_take_holiday_reduces_days(employee):
    employee.take_single_holiday()
    assert employee.vacation_days == 9

def test_payout_holiday_reduces_by_five(employee):
    employee.payout_holiday()
    assert employee.vacation_days == 5
```

Fixtures with broader reach go in `conftest.py` at the appropriate directory level. Pytest auto-discovers them.

### Fixture scope

```python
@pytest.fixture                       # function (default) — fresh per test
@pytest.fixture(scope="module")       # module — shared across the file
@pytest.fixture(scope="session")      # session — shared across the whole run
```

Default to `function`. Reach for wider scope only when setup is expensive (database connection, loaded ML model). **Never share mutable state across tests** — it makes failures order-dependent.

### Fixtures can depend on fixtures, and teardown via `yield`

```python
@pytest.fixture
def db_url() -> str:
    return "sqlite:///:memory:"

@pytest.fixture
def session(db_url):
    engine = create_engine(db_url)
    with Session(engine) as s:
        yield s
        s.rollback()
```

The `yield` form gives you teardown without `try/finally`.

---

## Fakes > Mocks

When you need to substitute a dependency, prefer a **fake** (a real, simpler implementation) over a **mock** (a spy that records calls).

```python
# BAD — mock; test couples to call shape, not behavior
def test_send_invoice():
    mailer = Mock()
    repo = Mock()
    repo.get.return_value = Invoice(id="1", customer_email="a@b.c", total=Decimal("10"))
    service = InvoiceService(repo=repo, mailer=mailer)

    service.send_invoice("1")

    mailer.send.assert_called_once_with("a@b.c", ANY)   # ANY hides what changed

# GOOD — fakes; test asserts on observable behavior
@dataclass
class FakeMailer:
    sent: list[tuple[str, str]] = field(default_factory=list)
    def send(self, to: str, body: str) -> None:
        self.sent.append((to, body))

@dataclass
class InMemoryInvoiceRepo:
    invoices: dict[str, Invoice] = field(default_factory=dict)
    def get(self, invoice_id: str) -> Invoice:
        return self.invoices[invoice_id]
    def add(self, inv: Invoice) -> None:
        self.invoices[inv.id] = inv

def test_send_invoice_emails_the_customer():
    repo = InMemoryInvoiceRepo()
    repo.add(Invoice(id="1", customer_email="a@b.c", total=Decimal("10")))
    mailer = FakeMailer()
    service = InvoiceService(repo=repo, mailer=mailer)

    service.send_invoice("1")

    assert mailer.sent == [("a@b.c", format_invoice(repo.get("1")))]
```

Fakes test behavior. Mocks test interactions. Behavior is what users see; interactions are an implementation detail. Reach for `Mock` only when the collaborator is genuinely external (a third-party SDK call you want to verify was issued).

---

## `monkeypatch` — for code you don't own

For replacing a function or attribute on an external object for the duration of one test.

```python
def test_load_data_handles_missing_file(monkeypatch, tmp_path):
    def fake_read(_):
        raise FileNotFoundError

    monkeypatch.setattr(pd, "read_csv", fake_read)

    with pytest.raises(DatasetNotFoundError):
        load_data(tmp_path / "missing.csv")
```

Don't monkeypatch your own code — that means the design is wrong. Fix it: inject the dependency instead. Reserve `monkeypatch` for third-party functions and environment variables (`monkeypatch.setenv("API_KEY", "test")`).

---

## Built-in fixtures worth knowing

- `tmp_path` — fresh temp directory per test (`Path`).
- `capsys` / `capfd` — capture stdout/stderr.
- `caplog` — capture log records.
- `monkeypatch` — patch attributes / env vars, auto-reverted.

```python
def test_logs_warning_on_retry_exhausted(caplog):
    with caplog.at_level(logging.WARNING):
        attempt_send_with_retry(broken_client)

    assert any("retry exhausted" in r.message for r in caplog.records)
```

---

## Async tests

Install `pytest-asyncio`. Either mark each test or enable auto mode.

```python
# Marker per test
@pytest.mark.asyncio
async def test_fetch_user_returns_user():
    user = await fetch_user(user_id=1)
    assert user.id == 1

# Or set once in pyproject.toml:
# [tool.pytest.ini_options]
# asyncio_mode = "auto"
```

For async fixtures:

```python
@pytest_asyncio.fixture
async def client() -> AsyncIterator[AsyncClient]:
    async with AsyncClient(app=app, base_url="http://test") as c:
        yield c
```

---

## Refactor for testability

If a function is hard to test, the function is wrong — not the test. Common fixes:

1. **Mixed I/O and computation** → split into pure + impure (recipe R21).
2. **Hardcoded dependency** → inject via constructor (recipe R26).
3. **God function** → split by responsibility (recipe R29).
4. **`time.time()` / `random` embedded** → inject `clock` and `rng`.

```python
# BAD — calls time.time() directly; cutoff logic is untestable in isolation
def is_session_expired(session: Session) -> bool:
    return time.time() - session.created_at > 3600

# GOOD — clock injected; tests pass fixed values
def is_session_expired(session: Session, now: float) -> bool:
    return now - session.created_at > 3600

def test_session_expired_after_one_hour():
    s = Session(created_at=1000.0)
    assert is_session_expired(s, now=4601.0) is True
    assert is_session_expired(s, now=4600.0) is False
```

The production call site does `is_session_expired(s, now=time.time())`. One line. Worth it.

---

## The test pyramid

- **Many unit tests** — fast, isolated, one function/class. The base of the pyramid.
- **Some integration tests** — two or three units cooperating (service + real repo against in-memory DB).
- **Few end-to-end tests** — running system through its real interface (HTTP, CLI). Slow; one per critical user flow.

If your suite is upside-down (mostly E2E, few unit), it's slow and flaky. Push tests down the pyramid: extract pure functions, fake the I/O layer, unit-test the core.

---

## What not to test

- **Private methods.** Test them through the public API. If you can't, the class is doing too much — split it.
- **Framework / library code.** Don't test that `dict.get` returns `None` for missing keys.
- **Trivial getters and dataclass construction.** `Order(id=1).id == 1` is a tautology.
- **`__str__` / `__repr__`** unless the format is part of a contract (CLI output, log format).

---

## What to test

- **Every public function:** at least one happy path, every distinct failure mode, edge cases (empty, zero, max, boundary).
- **Every branch your code introduces** — every `if`, every `match` arm, every `except`.
- **Every bug you fix:** write the failing test first, then the fix. The test is proof the bug existed and is now closed.

---

## Coverage

Aim for **80–90% line coverage** as a sanity check, not a target. 100% coverage with shallow tests is worse than 60% with sharp tests.

```toml
[tool.pytest.ini_options]
addopts = "-ra --strict-markers --cov=src --cov-report=term-missing --cov-fail-under=80"
```

`--cov-report=term-missing` prints uncovered lines. Read them — *that* is the signal, not the percentage. An untested branch in a critical money-handling function matters more than 100 untested lines of CLI argument plumbing.

---

## Property-based testing — `hypothesis`

For functions with invariants (parsers, encoders, math), supplement examples with property tests.

```python
from hypothesis import given, strategies as st

@given(st.text())
def test_encode_decode_roundtrips(s):
    assert decode(encode(s)) == s

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs):
    once = sorted(xs)
    twice = sorted(once)
    assert once == twice

@given(st.decimals(min_value=0, allow_nan=False, allow_infinity=False))
def test_tax_is_never_negative(price):
    v = VehicleInfo(brand="X", electric=False, catalogue_price=price)
    assert v.compute_tax() >= Decimal("0")
```

Hypothesis generates hundreds of inputs including edge cases you wouldn't think of (empty strings, Unicode, very large numbers). Reach for it when the function has a clean invariant ("roundtrips", "idempotent", "always non-negative") — not for simple equality assertions.

---

## Mutation testing — the ultimate check

If a test suite passes after a deliberately introduced bug, the suite is incomplete. Tools like `mutmut` mutate your source (flip `==` to `!=`, drop a `+ 1`, swap `and` for `or`) and report which mutations survive — i.e., which bugs your tests fail to catch.

```bash
mutmut run --paths-to-mutate src/billing/
mutmut results
```

Run it on critical modules (billing, auth, anything money- or security-related). 100% mutation kill rate is unrealistic; surviving mutants in core logic are bugs your tests missed.

---

## The discipline

- **Write the test before the fix** when you're patching a bug. The failing test is the spec.
- **Run tests on every commit.** Configure `pre-commit` to run `pytest -x` (stop on first failure).
- **Red is a stop-the-line event.** Don't push past failing tests; don't `xfail` to make CI green.
- **No `xfail` / `skip` without a comment** explaining why and when it can be removed. Skipped tests rot into permanent dead code.
- **A test you can't quickly explain is a test that's testing the wrong thing.** Rewrite or delete it.

---

## Quick reference

| Symptom | Fix |
|---|---|
| Using `unittest.TestCase` | Switch to plain functions + `assert` |
| Eight near-identical tests | `@pytest.mark.parametrize` |
| Shared setup duplicated | `@pytest.fixture` |
| `Mock` with `assert_called_with(ANY)` | Fake class, assert on captured state |
| Patching your own code | Inject the dependency instead |
| Untestable function | Split pure from impure (R21); inject collaborators (R26) |
| One test, many asserts | Split into one test per behavior |
| `assertEqual(str(e), "...")` | Domain exception with structured fields |
| `time.time()` inside the function | Inject `now: float` |
| Skipped test with no comment | Fix it or delete it |
