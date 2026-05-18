# Refactoring Recipes

Symptom → cure. Every entry is a code smell you'll actually encounter, paired with the specific fix.

Use this as a lookup table during code review or while writing. When you notice the smell, jump to the recipe.

---

## R1. `isinstance` chain dispatch → polymorphism

**Symptom:**
```python
def render(shape):
    if isinstance(shape, Circle):
        return f"○ r={shape.radius}"
    elif isinstance(shape, Square):
        return f"□ s={shape.side}"
    elif isinstance(shape, Triangle):
        return f"△ b={shape.base} h={shape.height}"
```

**Cure:** Move the behavior onto each class.

```python
class Shape(ABC):
    @abstractmethod
    def render(self) -> str: ...

@dataclass(frozen=True)
class Circle(Shape):
    radius: float
    def render(self) -> str: return f"○ r={self.radius}"

@dataclass(frozen=True)
class Square(Shape):
    side: float
    def render(self) -> str: return f"□ s={self.side}"
```

**Variant — when you can't add methods** (classes are third-party): use `match`:

```python
def render(shape: Shape) -> str:
    match shape:
        case Circle(radius=r): return f"○ r={r}"
        case Square(side=s):   return f"□ s={s}"
        case Triangle(base=b, height=h): return f"△ b={b} h={h}"
```

---

## R2. String-switch dispatch → Enum + dispatch dict

**Symptom:**
```python
def make_exporter(kind: str):
    if kind == "json": return JSONExporter()
    elif kind == "csv": return CSVExporter()
    elif kind == "xml": return XMLExporter()
    else: raise ValueError("unknown")
```

**Cure:**
```python
_EXPORTERS: dict[str, type[Exporter]] = {
    "json": JSONExporter,
    "csv": CSVExporter,
    "xml": XMLExporter,
}

def make_exporter(kind: str) -> Exporter:
    try:
        return _EXPORTERS[kind]()
    except KeyError:
        raise UnknownExporterError(kind) from None
```

---

## R3. Boolean flag parameter → two functions

**Symptom:**
```python
def take_holiday(self, payout: bool) -> None:
    if payout: ...
    else: ...
```

**Cure:** Split into two methods. Extract shared logic into a private helper.

```python
def payout_holiday(self) -> None: self._withdraw_holiday(5)
def take_single_holiday(self) -> None: self._withdraw_holiday(1)
def _withdraw_holiday(self, days: int) -> None: ...
```

---

## R4. Repeated `find_X` methods → one parameterized method

**Symptom:**
```python
def find_managers(self): return [e for e in self.employees if e.role == "manager"]
def find_interns(self): return [e for e in self.employees if e.role == "intern"]
def find_vps(self): return [e for e in self.employees if e.role == "vice_president"]
```

**Cure:**
```python
def find_employees(self, role: Role) -> list[Employee]:
    return [e for e in self.employees if e.role is role]
```

---

## R5. List scan with multiple keys → dict with tuple key

**Symptom:**
```python
def find_model(self, brand, model):
    for v in self.vehicles:
        if v.brand == brand and v.model == model:
            return v
    return None
```

**Cure:**
```python
self.vehicles: dict[tuple[str, str], VehicleInfo] = {}

def find_model(self, brand: str, model: str) -> VehicleInfo | None:
    return self.vehicles.get((brand, model))
```

---

## R6. `for ... append` → list comprehension

**Symptom:**
```python
result = []
for x in xs:
    if condition(x):
        result.append(transform(x))
```

**Cure:**
```python
result = [transform(x) for x in xs if condition(x)]
```

**Doesn't apply when:** the loop body does I/O, mutation, or anything with side effects. Comprehensions are for pure transformation.

---

## R7. Custom `to_string()` → `__str__`

**Symptom:**
```python
def get_info_str(self):
    return f"{self.brand} - {self.model}"

print(f"vehicle: {vehicle.get_info_str()}")
```

**Cure:**
```python
def __str__(self) -> str:
    return f"{self.brand} - {self.model}"

print(f"vehicle: {vehicle}")    # implicit __str__ call
```

---

## R8. Magic strings → Enum

**Symptom:**
```python
if order.status == "paid": ...
if order.status == "pending": ...
```

**Cure:**
```python
class OrderStatus(Enum):
    PAID = auto()
    PENDING = auto()
    REFUNDED = auto()

if order.status is OrderStatus.PAID: ...
```

---

## R9. Mutable dataclass with no real mutation → frozen

**Symptom:**
```python
@dataclass
class Customer:
    id: int
    name: str
    tier: str
```

**Cure:**
```python
@dataclass(frozen=True)
class Customer:
    id: int
    name: str
    tier: Tier
```

Now hashable, thread-safe, and free of "who mutated this?" bugs. Drop `frozen=True` only if you actually need to mutate.

---

## R10. Many positional args → config dataclass

**Symptom:**
```python
def run_experiment(data_path, output_dir, test_size, random_state, batch_size, epochs):
    ...

run_experiment("data.csv", "out/", 0.2, 42, 32, 10)   # which 0.2 means what?
```

**Cure:**
```python
@dataclass(frozen=True)
class TrainingConfig:
    data_path: Path
    output_dir: Path
    test_size: float = 0.2
    random_state: int = 42
    batch_size: int = 32
    epochs: int = 10

    def __post_init__(self) -> None:
        if not 0 < self.test_size < 1:
            raise ValueError(...)

def run_experiment(config: TrainingConfig) -> None: ...
```

---

## R11. Wildcard imports → explicit imports

**Symptom:**
```python
from utils import *
from constants import *
```

**Cure:**
```python
import utils                            # call as utils.thing
from constants import MAX_RETRIES, DEFAULT_TIMEOUT
```

---

## R12. Bare `except` / `except Exception` → specific exceptions

**Symptom:**
```python
try:
    do_thing()
except Exception:
    pass
```

**Cure:**
```python
try:
    do_thing()
except SpecificError as e:
    log.warning("expected: %s", e)
    # ... fallback or re-raise
```

If you don't know what specific exception can be raised, find out. "Catch everything" is never the right answer.

---

## R13. Generic `ValueError` → domain exception with context

**Symptom:**
```python
if days > self.vacation_days:
    raise ValueError("not enough days")
```

**Cure:**
```python
class NotEnoughVacationError(Exception):
    def __init__(self, requested: int, remaining: int) -> None:
        self.requested = requested
        self.remaining = remaining
        super().__init__(f"Requested {requested}, only {remaining} remaining")

if days > self.vacation_days:
    raise NotEnoughVacationError(requested=days, remaining=self.vacation_days)
```

---

## R14. Nested `if` → early return / guard clauses

**Symptom:**
```python
def process(user):
    if user is not None:
        if user.is_active:
            if user.has_permission("write"):
                do_thing(user)
            else:
                raise PermissionError
        else:
            raise InactiveError
    else:
        raise NotFoundError
```

**Cure:**
```python
def process(user: User | None) -> None:
    if user is None: raise NotFoundError
    if not user.is_active: raise InactiveError
    if not user.has_permission("write"): raise PermissionError
    do_thing(user)
```

Two indentation levels lighter; happy path reads top-to-bottom.

---

## R15. Comment labeling a code block → extracted function

**Symptom:**
```python
def process(payment_intent):
    # compute the cutoff timestamp
    ts = (datetime.now() - timedelta(hours=24)).timestamp()
    ...
    # determine the application fee
    charge = stripe.Charge.retrieve(payment_intent.latest_charge)
    bt = stripe.BalanceTransaction.retrieve(charge.balance_transaction)
    fee = bt.fee
    ...
```

**Cure:** Each labeling comment becomes a function name.

```python
def compute_cutoff_timestamp(hours_ago: int) -> float: ...
def get_application_fee(pi: stripe.PaymentIntent) -> int: ...

def process(payment_intent):
    ts = compute_cutoff_timestamp(24)
    fee = get_application_fee(payment_intent)
    ...
```

---

## R16. Loop body doing real work → extract per-iteration function

**Symptom:**
```python
for pi in payment_intents:
    if not is_processable(pi):
        continue
    data = construct_invoice_data(pi)
    invoice = create_and_send_invoice(data, InvoiceDelivery.EMAIL)
    book_invoice(invoice)
```

**Cure:**
```python
def process_payment_intent(pi: PaymentIntent) -> None:
    data = construct_invoice_data(pi)
    invoice = create_and_send_invoice(data, InvoiceDelivery.EMAIL)
    book_invoice(invoice)

for pi in successful_payment_intents:    # filter at the source
    process_payment_intent(pi)
```

---

## R17. Multiple loops over the same data → group once into a dict

**Symptom:**
```python
for customer in customers:
    customer_orders = [o for o in orders if o.customer_id == customer.id]  # O(n×m)
    ...
```

**Cure:**
```python
orders_by_customer: dict[int, list[Order]] = {}
for order in orders:
    orders_by_customer.setdefault(order.customer_id, []).append(order)

for customer in customers:
    customer_orders = orders_by_customer.get(customer.id, [])
    ...
```

O(n + m) instead of O(n × m).

---

## R18. Sentinel return value → `T | None` or domain exception

**Symptom:**
```python
def parse_int(s: str) -> int:
    try:
        return int(s)
    except ValueError:
        return -1                   # is -1 ever a real result? Forever unclear.
```

**Cure:**
```python
def parse_int(s: str) -> int | None:
    try:
        return int(s)
    except ValueError:
        return None
```

Or, if all callers want a specific failure: `def parse_int(s: str) -> int: ... raise ParseIntError(s)`.

---

## R19. Mutable default argument → `field(default_factory=...)` or `None`

**Symptom:**
```python
def add_item(item, items=[]):       # SHARED mutable default — bug!
    items.append(item)
    return items
```

**Cure:**
```python
def add_item(item, items: list | None = None) -> list:
    items = items if items is not None else []
    items.append(item)
    return items
```

In dataclasses:
```python
@dataclass
class Order:
    items: list[Item] = field(default_factory=list)
```

---

## R20. `print` for everything → structured logging

**Symptom:**
```python
print(f"processed user {user.id}")
print(f"ERROR: failed to send email to {user.email}")
```

**Cure:**
```python
log = logging.getLogger(__name__)

log.info("processed user", extra={"user_id": user.id})
log.error("email send failed", extra={"user_id": user.id, "email": user.email})
```

Queryable, filterable, doesn't get lost in stdout. (`print` is fine in scripts, CLIs talking to humans, and `__str__`-style demos.)

---

## R21. Mixed I/O and computation → split into pure + impure

**Symptom:**
```python
def calculate_and_save(orders, path):
    total = sum(o.total for o in orders if o.status is OrderStatus.PAID)
    path.write_text(json.dumps({"total": str(total)}))
```

**Cure:**
```python
def calculate_summary(orders: list[Order]) -> Summary:
    return Summary(total=sum(o.total for o in paid_orders(orders), Decimal("0")))

def save_summary(summary: Summary, path: Path) -> None:
    path.write_text(json.dumps({"total": str(summary.total)}))
```

`calculate_summary` is now pure → trivially testable.

---

## R22. `if x == True` / `if len(items) > 0` → idiomatic conditions

**Symptom:**
```python
if user.is_admin == True: ...
if len(items) > 0: ...
if some_var != None: ...
```

**Cure:**
```python
if user.is_admin: ...
if items: ...                 # truthy when non-empty
if some_var is not None: ...  # `is`, not `!=`, for None
```

---

## R23. Method that doesn't use `self` → static or module function

**Symptom:**
```python
class VehicleRegistry:
    def generate_id(self, length: int) -> str:
        return "".join(random.choices(string.ascii_uppercase, k=length))
```

**Cure:**
```python
class VehicleRegistry:
    @staticmethod
    def generate_id(length: int) -> str:
        return "".join(random.choices(string.ascii_uppercase, k=length))
```

Or move it to module level if it doesn't conceptually belong to the class.

---

## R24. Inheritance for shared behavior → composition

**Symptom:**
```python
class FileLogger:
    def log(self, msg): ...

class DBService(FileLogger):     # inheriting just to get .log
    def fetch(self): ...
```

**Cure:**
```python
@dataclass
class DBService:
    logger: Logger
    def fetch(self) -> ...:
        self.logger.log(...)
```

Inheritance is for "is-a". When you want "has-a" (logger, repository, formatter), use composition.

---

## R25. Long parameter list with several `Optional` → keyword-only + builder

**Symptom:**
```python
def create_user(email, password, name=None, age=None, role=None, tier=None, locale=None): ...
```

**Cure (light):** keyword-only with config dataclass:
```python
@dataclass(frozen=True)
class UserOptions:
    name: str | None = None
    age: int | None = None
    role: Role = Role.MEMBER
    tier: Tier = Tier.STANDARD
    locale: str = "en"

def create_user(email: str, password: str, *, options: UserOptions = UserOptions()): ...
```

**Cure (heavy):** builder pattern (see `05-design-patterns.md`).

---

## R26. Concrete dependency in constructor → inject the abstraction (DIP)

**Symptom:**
```python
class Switch:
    def __init__(self, bulb: LightBulb) -> None:    # tied to LightBulb forever
        self.bulb = bulb
    def press(self) -> None:
        self.bulb.turn_on()
```

**Cure:**
```python
class Switchable(Protocol):
    def turn_on(self) -> None: ...
    def turn_off(self) -> None: ...

class Switch:
    def __init__(self, device: Switchable) -> None:
        self.device = device
    def press(self) -> None:
        self.device.turn_on()

# Now any Switchable works:
Switch(LightBulb()).press()
Switch(Fan()).press()
Switch(FakeSwitchable()).press()   # tests
```

**Heuristic:** if your class names a concrete collaborator in its constructor and the test for that class needs to construct the real collaborator (database, HTTP client, third-party SDK), the dependency should be an abstraction.

---

## R27. Fat interface forcing empty / raising methods → split into focused Protocols (ISP)

**Symptom:**
```python
class PaymentProcessor(ABC):
    @abstractmethod
    def auth_sms(self, code: str) -> None: ...
    @abstractmethod
    def pay(self, order: Order) -> None: ...

class CreditPaymentProcessor(PaymentProcessor):
    def auth_sms(self, code: str) -> None:
        raise NotImplementedError("Credit cards don't support SMS")  # smell
    def pay(self, order: Order) -> None: ...
```

If a subclass has to raise `NotImplementedError` (or silently do nothing), the interface is too big.

**Cure:** split the interface along the lines of who needs what.

```python
class PaymentProcessor(Protocol):
    def pay(self, order: Order) -> None: ...

class SMSAuthenticator(Protocol):
    def auth_sms(self, code: str) -> None: ...

class CreditPaymentProcessor:
    def pay(self, order: Order) -> None: ...
    # no auth_sms — credit cards don't need it

@dataclass
class DebitPaymentProcessor:                          # implements both
    authenticator: SMSAuthenticator
    def auth_sms(self, code: str) -> None: self.authenticator.auth_sms(code)
    def pay(self, order: Order) -> None: ...
```

Now `CreditPaymentProcessor` only satisfies the interfaces it actually fulfils. Callers requiring SMS auth ask for `SMSAuthenticator`; callers requiring payment ask for `PaymentProcessor`. The type system enforces it.

---

## R28. Subclass that breaks the parent's contract → composition (LSP)

**Symptom:** A subclass changes inherited behavior in a way that surprises callers — different return type, narrower accepted input, raises where the parent doesn't.

```python
# Classic LSP violation: Square "is-a" Rectangle, but set_width breaks set_height's invariant
class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height
    def set_width(self, w: float) -> None:  self.width = w
    def set_height(self, h: float) -> None: self.height = h

class Square(Rectangle):
    def set_width(self, w: float) -> None:  self.width = self.height = w   # mutates height!
    def set_height(self, h: float) -> None: self.width = self.height = h

# Code that works for Rectangle now silently breaks for Square:
def expand(rect: Rectangle) -> None:
    rect.set_width(10)
    rect.set_height(5)
    assert rect.width == 10 and rect.height == 5     # fails for Square
```

**Cure:** drop the inheritance. Define a shared abstraction; both implement it independently.

```python
class Shape(Protocol):
    def area(self) -> float: ...

@dataclass(frozen=True)
class Rectangle:
    width: float
    height: float
    def area(self) -> float: return self.width * self.height

@dataclass(frozen=True)
class Square:
    side: float
    def area(self) -> float: return self.side ** 2
```

**Heuristic:** if a subclass's method has to violate the parent's documented contract — different exceptions, different invariants, different return shape — you wanted composition, not inheritance.

---

## R29. God class doing 5 things → extract by responsibility (cohesion)

**Symptom:**
```python
class Application:
    def register_vehicle(self, brand: str) -> None:
        # generate id
        vehicle_id = "".join(random.choices(string.ascii_uppercase, k=12))
        # generate license plate
        license_plate = f"{vehicle_id[:2]}-{...}-{...}"
        # look up price
        if brand == "Tesla Model 3":   catalogue_price = 60_000
        elif brand == "Volkswagen ID3": catalogue_price = 35_000
        elif brand == "BMW 5":          catalogue_price = 45_000
        # compute tax
        tax_percentage = 0.02 if brand in {"Tesla Model 3", "Volkswagen ID3"} else 0.05
        payable_tax = tax_percentage * catalogue_price
        # print
        print(f"Brand: {brand}, id: {vehicle_id}, plate: {license_plate}, tax: {payable_tax}")
```

Five comment-labelled sections in one method — five responsibilities.

**Cure:** every comment becomes a class or function with a name.

```python
@dataclass(frozen=True)
class VehicleInfo:
    brand: str
    electric: bool
    catalogue_price: Decimal
    def compute_tax(self) -> Decimal:
        rate = Decimal("0.02") if self.electric else Decimal("0.05")
        return rate * self.catalogue_price

@dataclass(frozen=True)
class Vehicle:
    id: str
    license_plate: str
    info: VehicleInfo

class VehicleRegistry:
    def __init__(self) -> None:
        self._catalog: dict[str, VehicleInfo] = {...}
    def create_vehicle(self, brand: str) -> Vehicle:
        return Vehicle(id=generate_id(12), license_plate=generate_license(), info=self._catalog[brand])

class Application:
    def __init__(self, registry: VehicleRegistry) -> None:
        self.registry = registry
    def register_vehicle(self, brand: str) -> Vehicle:
        return self.registry.create_vehicle(brand)
```

Each class is independently testable. `VehicleInfo.compute_tax()` is a pure function of three fields. `Application` becomes a one-line orchestrator.

---

## Quick lookup table

| Smell | Recipe |
|---|---|
| `isinstance` chain | R1 |
| String-switch | R2 |
| `flag: bool` param | R3 |
| Repeated `find_X` | R4 |
| Linear search by composite key | R5 |
| `for ... append` | R6 |
| Custom `to_string()` | R7 |
| Magic strings | R8 |
| Mutable dataclass without need | R9 |
| 4+ positional args | R10 |
| Wildcard imports | R11 |
| `except Exception:` | R12 |
| Generic `ValueError("msg")` | R13 |
| Nested `if` ladders | R14 |
| Comment labeling a block | R15 |
| Real work in a loop body | R16 |
| O(n×m) loop joins | R17 |
| Sentinel return values | R18 |
| Mutable default arg | R19 |
| `print` for production logs | R20 |
| Mixed I/O + computation | R21 |
| `== True`, `len > 0`, `!= None` | R22 |
| `self`-less method | R23 |
| Inheritance for shared util | R24 |
| Big optional param list | R25 |
| Concrete dependency in constructor (DIP) | R26 |
| Fat interface forcing empty methods (ISP) | R27 |
| Subclass breaks parent contract (LSP) | R28 |
| God class with 5 responsibilities | R29 |
