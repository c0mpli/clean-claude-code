# Design Patterns

When and how to apply each pattern, with Python-native implementations. Most patterns are simpler in Python than in the GoF book — lean into language features (dataclasses, Protocol, first-class functions) instead of porting Java verbatim.

---

## Repository

**Problem:** Business logic shouldn't know whether data lives in SQLite, Postgres, an in-memory dict, or a flat file.

**Pattern:** A generic abstract `Repository[T]` interface with CRUD methods. Concrete subclasses implement against specific storage. Business code depends on the interface.

```python
class Repository[T](ABC):
    @abstractmethod
    def get(self, id: int) -> T: ...
    @abstractmethod
    def list_all(self) -> list[T]: ...
    @abstractmethod
    def add(self, entity: T) -> None: ...
    @abstractmethod
    def remove(self, id: int) -> None: ...


class PostRepository(Repository[Post]):
    def __init__(self, db_path: str) -> None:
        self.db_path = db_path

    def get(self, id: int) -> Post:
        with self._connect() as cur:
            cur.execute("SELECT * FROM posts WHERE id=?", (id,))
            row = cur.fetchone()
            if row is None:
                raise PostNotFoundError(id)
            return Post(*row)

    def list_all(self) -> list[Post]:
        with self._connect() as cur:
            cur.execute("SELECT * FROM posts")
            return [Post(*row) for row in cur.fetchall()]
```

**When to use:** Whenever you have more than one consumer of the data, or you anticipate swapping storage (e.g., in-memory for tests, Postgres in prod).

**When not to use:** Throwaway scripts. A single SQL function may not need wrapping.

---

## Factory

**Problem:** Selecting and constructing one of several related objects based on configuration.

**Pattern:** Either a function that returns by a key, or an abstract factory class for families of related objects.

### Simple factory function

```python
class Exporter(Protocol):
    def export(self, path: Path) -> None: ...

_EXPORTERS: dict[str, type[Exporter]] = {
    "json": JSONExporter,
    "csv": CSVExporter,
    "parquet": ParquetExporter,
}

def make_exporter(kind: str) -> Exporter:
    try:
        return _EXPORTERS[kind]()
    except KeyError:
        raise UnknownExporterError(kind) from None
```

### Abstract factory (for related object families)

```python
class ExporterFactory(ABC):
    @abstractmethod
    def get_video_exporter(self) -> VideoExporter: ...
    @abstractmethod
    def get_audio_exporter(self) -> AudioExporter: ...


class FastExporter(ExporterFactory):
    def get_video_exporter(self) -> VideoExporter: return H264BPVideoExporter()
    def get_audio_exporter(self) -> AudioExporter: return AACAudioExporter()


class HighQualityExporter(ExporterFactory):
    def get_video_exporter(self) -> VideoExporter: return H264Hi422PVideoExporter()
    def get_audio_exporter(self) -> AudioExporter: return AACAudioExporter()


_FACTORIES = {
    "fast": FastExporter,
    "hq": HighQualityExporter,
}

def read_factory(name: str) -> ExporterFactory:
    return _FACTORIES[name]()
```

**Key insight:** The `if/elif` chain disappears. Each factory owns its family's construction rules. The selection table is the one place that names them all.

---

## Strategy (via Protocol)

**Problem:** Multiple interchangeable algorithms for the same task (sorting, pricing, notification delivery).

**Pattern:** A `Protocol` describes the algorithm's shape. Each algorithm is a class (or function). The user injects which strategy to use.

```python
class PricingStrategy(Protocol):
    def price(self, items: list[Item]) -> Decimal: ...


@dataclass
class StandardPricing:
    def price(self, items: list[Item]) -> Decimal:
        return sum((i.unit_price * i.qty for i in items), Decimal("0"))


@dataclass
class PremiumPricing:
    discount: Decimal = Decimal("0.1")
    def price(self, items: list[Item]) -> Decimal:
        subtotal = sum((i.unit_price * i.qty for i in items), Decimal("0"))
        return subtotal * (Decimal("1") - self.discount)


def checkout(cart: Cart, pricing: PricingStrategy) -> Decimal:
    return pricing.price(cart.items)
```

**Functional alternative:** When the strategy is a single function, just pass the function. No class needed.

```python
def checkout(cart: Cart, price: Callable[[list[Item]], Decimal]) -> Decimal:
    return price(cart.items)

checkout(cart, standard_price)
checkout(cart, lambda items: premium_price(items, discount=Decimal("0.15")))
```

**Heuristic:** If the strategy has state or multiple methods, use a class. If it's one function, pass a function.

---

## Builder (fluent)

**Problem:** Constructing an object with many optional fields, where the constructor would balloon to 15 keyword arguments.

**Pattern:** A builder class with chainable setters returning `Self`, and a final `.build()` that produces the immutable result.

```python
class HTMLBuilder:
    def __init__(self) -> None:
        self._title: str = "Untitled"
        self._body: list[str] = []
        self._meta: dict[str, str] = {}

    def set_title(self, title: str) -> Self:
        self._title = title
        return self

    def add_header(self, text: str, level: int = 1) -> Self:
        self._body.append(f"<h{level}>{text}</h{level}>")
        return self

    def add_paragraph(self, text: str) -> Self:
        self._body.append(f"<p>{text}</p>")
        return self

    def set_meta(self, key: str, value: str) -> Self:
        self._meta[key] = value
        return self

    def build(self) -> HTMLPage:
        return HTMLPage(title=self._title, meta=self._meta, body=self._body)


page = (
    HTMLBuilder()
        .set_title("Welcome")
        .add_header("Hello")
        .add_paragraph("Built with the builder pattern.")
        .set_meta("description", "Demo page")
        .build()
)
```

**Why `Self`:** Subclasses of `HTMLBuilder` get correct return types automatically. `-> HTMLBuilder` would break inheritance.

**When not to use:** If the object has fewer than ~4 optional fields, just use keyword arguments to the constructor.

---

## Dependency Injection

**Problem:** A service needs a database connection / HTTP client / config — but constructing it inline makes testing painful and couples the service to a specific implementation.

**Pattern level 1 — constructor injection (lightweight, no framework):**

```python
class InvoiceService:
    def __init__(self, repo: InvoiceRepository, mailer: Mailer) -> None:
        self.repo = repo
        self.mailer = mailer

    def send_invoice(self, invoice_id: str) -> None:
        invoice = self.repo.get(invoice_id)
        self.mailer.send(invoice.customer_email, format(invoice))

# Production
service = InvoiceService(repo=PostgresInvoiceRepo(), mailer=SMTPMailer(...))

# Test
service = InvoiceService(repo=InMemoryInvoiceRepo(), mailer=FakeMailer())
```

This is the default. 90% of "DI" needs are solved by passing dependencies to constructors.

**Pattern level 2 — DI framework (when wiring gets nested):**

```python
import inject

@inject.params(cursor=sqlite3.Cursor)
def create_post(post: Post, cursor: sqlite3.Cursor) -> Post:
    cursor.execute("INSERT INTO posts(title, content) VALUES(?, ?)", (post.title, post.content))
    cursor.connection.commit()
    return post

def configure_di(db: sqlite3.Connection) -> None:
    inject.configure(
        lambda binder: binder
            .bind_to_constructor(sqlite3.Connection, lambda: db)
            .bind_to_provider(sqlite3.Cursor, db.cursor)
    )
```

**When to reach for a framework:** Many services with overlapping dependencies, scopes (request-scoped, singleton, etc.), or you're tired of constructor explosion. For most projects, constructor injection is enough.

---

## CQRS (Command Query Responsibility Segregation)

**Problem:** Your write model and read model have diverged. Reads need precomputed fields (preview text, denormalized counts), but storing those on the write model means every command has to update them.

**Pattern:** Two stores. The write store is the source of truth. A projector builds an optimized read store from write events. Queries hit only the read store.

```python
COMMANDS_COLL = "ticket_commands"
READS_COLL = "ticket_reads"


async def cmd_create_ticket(db: Database, cmd: CreateTicket) -> str:
    doc = {"subject": cmd.subject, "message": cmd.message, "status": "open", ...}
    res = await db[COMMANDS_COLL].insert_one(doc)
    await project_ticket(db, str(res.inserted_id))
    return str(res.inserted_id)


async def project_ticket(db: Database, ticket_id: str) -> None:
    doc = await db[COMMANDS_COLL].find_one({"_id": ObjectId(ticket_id)})
    read_doc = {
        "_id": doc["_id"],
        "subject": doc["subject"],
        "status": doc["status"],
        "preview": make_preview(doc.get("message", "")),
        "has_note": bool((doc.get("agent_note") or "").strip()),
    }
    await db[READS_COLL].update_one(
        {"_id": doc["_id"]}, {"$set": read_doc}, upsert=True
    )


# Query side — trivial, fast, no computation
@app.get("/tickets")
async def list_tickets(has_note: bool | None = None) -> list[TicketListItem]:
    query = {} if has_note is None else {"has_note": has_note}
    cursor = db[READS_COLL].find(query)
    return [TicketListItem(**doc) async for doc in cursor]
```

**When to use:** Reads are expensive and frequent. Read shape diverges meaningfully from write shape. You want to filter/index on computed fields.

**When not to use:** Reads are cheap. Models are nearly identical. (CQRS adds operational complexity — eventual consistency, projector failures.)

---

## Unit of Work

**Problem:** A use case touches several repositories and should commit-or-rollback as one transaction.

**Pattern:** A context manager that owns the transaction; repositories share its connection.

```python
class UnitOfWork:
    def __init__(self, session_factory: Callable[[], Session]) -> None:
        self._session_factory = session_factory

    def __enter__(self) -> Self:
        self.session = self._session_factory()
        self.invoices = InvoiceRepository(self.session)
        self.payments = PaymentRepository(self.session)
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        if exc_type is None:
            self.session.commit()
        else:
            self.session.rollback()
        self.session.close()


# Usage
with UnitOfWork(SessionLocal) as uow:
    invoice = uow.invoices.get(invoice_id)
    payment = Payment(invoice_id=invoice.id, amount=invoice.total)
    uow.payments.add(payment)
    invoice.mark_paid()
    # commit on __exit__, rollback if anything raised
```

**When to use:** Multi-repository transactions. Replaces sprinkling `db.begin() / db.commit() / db.rollback()` across the codebase.

---

## Observer / Pub-Sub

**Problem:** Many parts of the system care when something happens (`order.paid`, `user.signed_up`), but the source shouldn't know about all of them.

**Pattern — simple in-process:**

```python
class EventBus:
    def __init__(self) -> None:
        self._handlers: dict[type, list[Callable]] = defaultdict(list)

    def subscribe[E](self, event_type: type[E], handler: Callable[[E], None]) -> None:
        self._handlers[event_type].append(handler)

    def publish[E](self, event: E) -> None:
        for handler in self._handlers[type(event)]:
            handler(event)


@dataclass(frozen=True)
class OrderPaid:
    order_id: str
    amount: Decimal
    paid_at: datetime


bus = EventBus()
bus.subscribe(OrderPaid, send_receipt_email)
bus.subscribe(OrderPaid, record_revenue_metric)

bus.publish(OrderPaid(order_id="x", amount=Decimal("9.99"), paid_at=now()))
```

**Distributed case:** Use a real message bus (RabbitMQ, Kafka, Redis Streams). The shape stays the same; the bus implementation changes.

---

## Decision summary

| If you need... | Use... |
|---|---|
| Abstract over storage backends | Repository |
| Choose one of several similar objects from config | Factory function |
| Build families of related objects | Abstract factory |
| Swap algorithm at runtime | Strategy via Protocol (or a Callable) |
| Construct objects with many optional fields | Fluent builder returning `Self` |
| Inject collaborators | Constructor injection. Framework only if it gets nested |
| Transactional multi-repo writes | Unit of Work |
| Decouple producers from consumers | Event bus |
| Reads diverge from writes | CQRS |

When in doubt, **don't** reach for a pattern. The simplest thing that works — a function, a dataclass, a dict — is usually right. Patterns are for when the simple thing has actually become painful.
