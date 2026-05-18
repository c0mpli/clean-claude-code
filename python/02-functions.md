# Function Design

The single most important file in this guide. Function design is where most code goes wrong, and where Arjan's style differs most strikingly from typical Python.

## The core rule

> **A function does one thing. If you can describe what it does without using the word "and", it has one responsibility. Otherwise, split it.**

The split usually adds lines and adds names. That's the point. Each piece becomes independently testable, independently readable, and — when shared logic exists between siblings — extractable into a private helper that *prevents* duplication rather than causing it.

## When to split

Split a function when **any** of these are true:

1. You wrote a comment that labels a block ("# now we validate the input"). The block wants a name.
2. The function has a `flag: bool` parameter that changes its behavior.
3. The function has more than ~15 statements, OR nests more than 2 levels deep.
4. You can describe its behavior using the word "and" (or "then", or "next").
5. Two sibling functions (in the same class or module) share a 3+ line block.
6. The function loops over a collection AND does non-trivial work on each element. (Split the per-element work into its own function.)
7. You feel the urge to write a docstring longer than one line because the function does too much.

## How to split — five concrete patterns

### Pattern A: Boolean-flag method → two methods + private helper

The most common smell. A boolean argument is two methods pretending to be one.

```python
# BAD
def take_holiday(self, payout: bool) -> None:
    if payout:
        if self.vacation_days < 5:
            raise ValueError(f"Not enough days: {self.vacation_days}")
        self.vacation_days -= 5
        print(f"Paid out. Days left: {self.vacation_days}")
    else:
        if self.vacation_days < 1:
            raise ValueError("No days left")
        self.vacation_days -= 1
        print("Have fun!")

# GOOD
def payout_holiday(self) -> None:
    self._withdraw_holiday(5)
    print(f"Paid out. Days left: {self.vacation_days}")

def take_single_holiday(self) -> None:
    self._withdraw_holiday(1)
    print("Have fun!")

def _withdraw_holiday(self, days: int) -> None:
    if self.vacation_days < days:
        raise NotEnoughVacationError(requested=days, remaining=self.vacation_days)
    self.vacation_days -= days
```

Why this is better:
- Call sites: `employee.take_single_holiday()` — clear. Versus `employee.take_holiday(False)` — what does `False` mean?
- Each public method has one job. The shared invariant check and decrement is in `_withdraw_holiday`. No duplication.
- The private helper enforces an invariant (`vacation_days >= days`) in exactly one place.

### Pattern B: Nested loops → one function per concern

When a function has nested loops doing different conceptual work at each level, each level usually wants its own function.

```python
# BAD — group, filter, sum, discount, aggregate all tangled
def generate_customer_report(customers, orders):
    report = []
    for customer in customers:
        total = Decimal("0")
        count = 0
        for order in orders:
            if order.customer_id == customer.id:
                if order.status == "paid":
                    order_total = Decimal("0")
                    for item in order.items:
                        order_total += item.price * item.quantity
                    if customer.tier == "premium" and order_total > 100:
                        order_total *= Decimal("0.9")
                    total += order_total
                    count += 1
        if count > 0:
            report.append(CustomerSummary(customer.name, count, total))
    return report

# GOOD
def group_orders_by_customer(orders: list[Order]) -> dict[int, list[Order]]:
    grouped: dict[int, list[Order]] = {}
    for order in orders:
        grouped.setdefault(order.customer_id, []).append(order)
    return grouped

def paid_orders(orders: list[Order]) -> list[Order]:
    return [o for o in orders if o.status is OrderStatus.PAID]

def apply_discount(customer: Customer, total: Decimal) -> Decimal:
    if customer.tier is Tier.PREMIUM and total > Decimal("100"):
        return total * Decimal("0.9")
    return total

def build_customer_summary(customer: Customer, orders: list[Order]) -> CustomerSummary:
    totals = [apply_discount(customer, o.total) for o in paid_orders(orders)]
    return CustomerSummary(
        customer_name=customer.name,
        paid_orders=len(totals),
        total_spent=sum(totals, Decimal("0")),
    )

def generate_customer_report(customers, orders) -> list[CustomerSummary]:
    by_customer = group_orders_by_customer(orders)
    return [
        build_customer_summary(c, by_customer[c.id])
        for c in customers
        if c.id in by_customer
    ]
```

Bonus: `order.total` moves onto the `Order` dataclass as a `@property` — the "sum the items" loop disappears from the report code entirely.

Note the line count *went up*. Worth it: `apply_discount` is now independently unit-testable. Reading any single function fits in your head. The top-level function reads like a recipe.

### Pattern C: Pull "one iteration's work" out of a loop

When a `for` loop's body is non-trivial, extract the body. The loop's responsibility is *to iterate*; the body's responsibility is *to process one*.

```python
# BAD
for pi in payment_intents:
    if not is_processable(pi):
        continue
    invoice_data = construct_invoice_data(pi)
    invoice = create_and_send_invoice(invoice_data, InvoiceDelivery.EMAIL)
    book_invoice(invoice)

# GOOD
def process_payment_intent(pi: stripe.PaymentIntent) -> None:
    invoice_data = construct_invoice_data(pi)
    invoice = create_and_send_invoice(invoice_data, InvoiceDelivery.EMAIL)
    book_invoice(invoice)

for pi in successful_payment_intents:   # filter at the source, not inside
    process_payment_intent(pi)
```

Also notice: the `if not is_processable(pi): continue` is gone. Filter at the *source* (rename the variable / function that produces the list to return only successful ones), don't filter mid-loop.

### Pattern D: Comment-labeled block → named function

A comment that labels a block of code is a function name asking to be born.

```python
# BAD
def construct_invoice(pi):
    # compute the cutoff timestamp
    timestamp = (datetime.now() - timedelta(hours=24)).timestamp()
    ...
    # determine the application fee
    charge = stripe.Charge.retrieve(pi.latest_charge)
    if not charge: raise ValueError(...)
    bt = stripe.BalanceTransaction.retrieve(charge.balance_transaction)
    if not bt: raise ValueError(...)
    fee = bt.fee
    ...

# GOOD
def compute_cutoff_timestamp(hours_ago: int) -> float:
    return (datetime.now() - timedelta(hours=hours_ago)).timestamp()

def get_application_fee(pi: stripe.PaymentIntent) -> int:
    charge = stripe.Charge.retrieve(pi.latest_charge)
    if not charge:
        raise ChargeNotFoundError(pi.latest_charge)
    bt = stripe.BalanceTransaction.retrieve(charge.balance_transaction)
    if not bt:
        raise BalanceTransactionNotFoundError(charge.balance_transaction)
    return bt.fee

def construct_invoice(pi):
    timestamp = compute_cutoff_timestamp(24)
    fee = get_application_fee(pi)
    ...
```

The comments are gone because the function names *are* the comments. Code that needs a comment to explain what a block does is asking to be extracted.

### Pattern E: Pipeline → labeled sections, one function per stage

For multi-stage pipelines (data processing, ML training, ETL), use comment dividers and one function per stage. The orchestrator becomes a recipe.

```python
# --------------------------- Data loading ---------------------------
def load_data(path: Path) -> pd.DataFrame: ...

# --------------------------- Preprocessing ---------------------------
def clean_data(df: pd.DataFrame) -> pd.DataFrame: ...
def engineer_features(df: pd.DataFrame) -> pd.DataFrame: ...

# --------------------------- Training ---------------------------
def build_model(seed: int) -> Model: ...
def train_model(model: Model, X, y) -> None: ...

# --------------------------- Evaluation ---------------------------
def evaluate_model(model: Model, X, y) -> Metrics: ...

# --------------------------- Persistence ---------------------------
def save_artifacts(config: TrainingConfig, model, metrics) -> None: ...

# --------------------------- Orchestration ---------------------------
def run_experiment(config: TrainingConfig) -> None:
    df = load_data(config.data_path)
    df = clean_data(df)
    df = engineer_features(df)
    X, y = split_features_and_target(df)
    X_train, X_test, y_train, y_test = split_data(X, y, test_size=config.test_size)
    model = build_model(config.random_state)
    train_model(model, X_train, y_train)
    metrics = evaluate_model(model, X_test, y_test)
    save_artifacts(config, model, metrics)
```

`train_model` may be literally `model.fit(X_train, y_train)` — a one-liner. Wrap it anyway. The name gives it a place in the recipe, and now tests can mock it.

## Naming

Functions: **verb + object**. Variables: **noun**. No abbreviations. No type info in the name (the type hint already says it).

```python
# Good
calculate_total_price(items, discount)
fetch_user_by_id(user_id)
is_valid_email(address)        # predicate — starts with is_/has_/can_
to_json(data)                  # converter — starts with to_

# Bad
total(items, d)                    # too vague
calculate_integer_total_price(...) # type info in name
get_data()                         # what data?
process()                          # process what, how?
```

Private functions start with `_`. Module-private helpers exist freely; classes don't need to hold every function.

## Docstrings

Docstrings are not free. They rot, they lie, and they crowd the screen. Write them when the function has a **contract** worth stating — not by default.

### Rule: docstring iff the function is public AND non-trivial

A docstring exists when **both** are true:
1. The function is part of a public API (importable from a package, exposed via HTTP/CLI, called by code you don't control).
2. There's a contract beyond the signature: side effects, preconditions, invariants, complexity, units, exceptions raised.

If the signature + name + body fit on one screen and behave as expected, **no docstring**.

```python
# NO docstring — name + types say everything
def total_price(items: list[Item]) -> Decimal:
    return sum((i.unit_price * i.qty for i in items), Decimal("0"))

# NO docstring — private helper
def _next_id(self) -> int:
    return max((t.id for t in self.repo.list_all()), default=0) + 1
```

### Document WHY, contracts, side effects — never WHAT

The type hints already say what the parameters are. The body already says what the function does. The docstring's job is the part neither captures.

```python
# BAD — restates the obvious; rots the moment behavior changes
def compute_tax(price: Decimal, rate: Decimal) -> Decimal:
    """
    Compute tax.

    Args:
        price: The price.
        rate: The rate.

    Returns:
        The tax.
    """
    return price * rate

# GOOD — no docstring needed. Name says it; types say it.
def compute_tax(price: Decimal, rate: Decimal) -> Decimal:
    return price * rate

# GOOD — docstring states the contract that types can't
def transfer(from_account: Account, to_account: Account, amount: Money) -> Transfer:
    """Atomically debit from_account and credit to_account.

    Acquires both account locks in id order to prevent deadlock. Rolls
    back the entire transfer if either side fails. Idempotent on retry
    when called with the same (from, to, amount, request_id) tuple within
    24 hours.

    Raises:
        InsufficientFundsError: from_account.balance < amount.
        CurrencyMismatchError: account currencies differ from amount.currency.
    """
    ...
```

### Format — Google style, one-line for most

Use Google style. It's the cleanest for both humans and LLMs.

```python
def fetch_user(user_id: int, *, timeout_s: float = 30.0) -> User:
    """Fetch a user by ID from the upstream identity service.

    Args:
        user_id: Internal user ID, not the external auth provider ID.
        timeout_s: Per-request timeout. Retries are not attempted at this layer.

    Returns:
        The fully hydrated User including roles and tenant membership.

    Raises:
        UserNotFoundError: No user exists with this ID.
        IdentityServiceUnavailableError: Upstream returned 5xx or timed out.
    """
    ...
```

For most functions, **one line is enough**:

```python
def parse_iso_date(s: str) -> date:
    """Parse an ISO 8601 date. Raises ValueError on malformed input."""
    ...
```

Skip the `Args:` / `Returns:` / `Raises:` sections when their content would just be "the same thing the signature already says".

### Module and class docstrings

Module-level docstring: one paragraph at the top, only if the module isn't self-explanatory from its name. `src/billing/invoice.py` doesn't need "This module is about invoices."

Class docstring: state the **role** of the class — what kind of thing it is and what invariants it maintains. Not a list of its methods (the reader can see those).

```python
class InvoiceService:
    """Use cases for invoices: create, send, void, and refund.

    Construction requires an InvoiceRepository and a Mailer. Operations
    are transactional via the repository's Unit of Work; partial failures
    roll back. Not thread-safe — instantiate per request.
    """
```

### Anti-patterns

- **Restating type hints in `Args:`.** `customer_id: Customer's id (int).` — delete.
- **Mismatched docstring and behavior.** The docstring says "returns None on missing" but the code raises. Now the docstring is a lie. Either delete it or fix one of them.
- **Multi-paragraph docstrings on three-line functions.** The function is small enough to read. The docstring is bigger than the function. Delete.
- **Sphinx `:param X:` / `:returns:` syntax.** Verbose, harder to read at a glance than Google style. Pick Google.
- **Docstrings on every test function.** Tests are named for what they assert. `def test_compute_tax_negative_exemption_raises(): ...` is the docstring.

### The check

If you delete the docstring and a competent reader can still answer "what does this do, what does it assume, what does it promise" from the signature and body — the docstring was noise.

---

## Function size

There's no hard rule, but in practice:

- **Most functions: 3–10 lines.**
- **Orchestrators: up to ~20 lines, but should read top-to-bottom like a recipe** — no nested logic.
- **More than 20 lines? Split.** Find the seams.
- **More than 2 levels of indentation? Split, or use early returns.**

## Arguments

- **Type-hint every parameter.** Even `self` doesn't need it; everything else does.
- **No mutable default arguments.** Use `field(default_factory=list)` in dataclasses or `None` + assignment.
- **Group 4+ related arguments into a config dataclass.** Adding a knob shouldn't require touching every caller.
- **Use keyword-only arguments (`*,`) for clarity-critical parameters.**

```python
# GOOD — keyword-only forces clarity at the call site
def split_data(X, y, *, test_size: float, random_state: int) -> tuple[...]:
    return train_test_split(X, y, test_size=test_size, random_state=random_state)

# Call site: must use keywords, can't misorder
X_train, X_test, y_train, y_test = split_data(X, y, test_size=0.2, random_state=42)
```

## Return values

- **Always type-hint the return.** Use `-> None` explicitly when nothing is returned. (It's not assumed.)
- **Return early on guard conditions.** Don't wrap the happy path in `else`.
- **One return type, not "either dict or None or False".** If the function can fail, use `T | None` or `Result[T, E]`, not sentinel values.
- **For builders / fluent APIs, return `Self`.**

```python
# BAD
def find_user(user_id: int):           # no return type
    user = db.get(user_id)
    if user is None:
        return False                   # mixed return type
    return user

# GOOD
def find_user(user_id: int) -> User | None:
    return db.get(user_id)
```

## Pure functions where possible

- Functions that take inputs and return outputs (no I/O, no mutation) are the easiest to test.
- Push I/O to the edges of your call graph. The inside should be pure.
- If a function both *computes* and *writes to disk*, split it.

```python
# BAD
def calculate_and_save_report(orders, path):
    total = sum(o.total for o in orders if o.status is OrderStatus.PAID)
    summary = Summary(total=total, count=len(orders))
    path.write_text(json.dumps(asdict(summary)))

# GOOD
def calculate_summary(orders: list[Order]) -> Summary:
    paid = paid_orders(orders)
    return Summary(total=sum(o.total for o in paid), count=len(paid))

def save_summary(summary: Summary, path: Path) -> None:
    path.write_text(json.dumps(asdict(summary)))
```

Now `calculate_summary` is trivially unit-testable.

## The "this is more code!" objection

Yes. The trade is explicit:

| Cost | Benefit |
|---|---|
| More function definitions | Each is independently testable |
| More names | Names *replace* comments and scrolling — read the orchestrator, dive only where needed |
| Indirection | Cyclomatic complexity per function drops; nothing has more than 1–2 branches |
| Risk of duplication | *Prevented* by private helpers — splitting gives shared logic a place to live |

Read the orchestrator first. If you need to know how a step works, click into the function. You almost never need to. That's the win.

## The heuristic

> **If you can describe a function without using the word "and", it has one responsibility.**

The moment you hear yourself saying "this loads data **and** validates it **and** writes it" — split it.
