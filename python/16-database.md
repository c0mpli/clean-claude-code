# Databases

How to talk to a database without slowing down, corrupting data, or rewriting the world when you need a second index. Examples use SQLAlchemy 2.x; the rules apply to any ORM and to raw SQL.

---

## The N+1 query — most common bug in any app

Loading a collection and then triggering one query per item to fetch a related field.

```python
# BAD — 1 query for orders, then 1 per order to fetch customer = N+1 queries
orders = session.scalars(select(Order)).all()
for order in orders:
    print(f"{order.id}: {order.customer.email}")          # SELECT customer per iteration
```

For 100 orders, that's 101 queries. At 1ms each, ~100ms of latency that should have been 2ms.

```python
# GOOD — eager-load the relationship in one query
from sqlalchemy.orm import selectinload, joinedload

orders = session.scalars(
    select(Order).options(selectinload(Order.customer))
).all()
for order in orders:
    print(f"{order.id}: {order.customer.email}")          # 0 extra queries
```

| Strategy | When to use |
|---|---|
| `selectinload` | One-to-many or many-to-many. Issues a second `WHERE id IN (...)` query. Avoids row explosion. |
| `joinedload` | Many-to-one or one-to-one. Single `JOIN`. Risks row duplication on collections. |
| `subqueryload` | Older alternative to `selectinload`. Prefer `selectinload`. |
| `raiseload` | "I forgot to eager-load this and I want to know." Raises on lazy-load. |

**The discipline:** if a request hits the same query template more than once, you have an N+1. Configure your ORM to log SQL in dev (SQLAlchemy: `echo=True`) and watch the count.

---

## Indexes — every FK, every WHERE column

The rule: an index on every column that appears in `WHERE`, `JOIN`, or `ORDER BY`. Foreign keys especially — most ORMs don't create them automatically.

```python
class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[int] = mapped_column(ForeignKey("customers.id"), index=True)
    status: Mapped[OrderStatus] = mapped_column(index=True)
    created_at: Mapped[datetime] = mapped_column(index=True)

    __table_args__ = (
        Index("ix_orders_customer_status", "customer_id", "status"),    # composite
    )
```

**Composite indexes:** column order matters. `(customer_id, status)` can serve queries on `customer_id` alone *and* `customer_id + status`, but **not** `status` alone. Put the most-selective / most-filtered column first.

**Don't over-index.** Every index slows writes and consumes disk. Profile slow queries (`EXPLAIN`) before adding, not as a precaution.

**Drop indexes you don't use.** PostgreSQL: `pg_stat_user_indexes`. MySQL: `sys.schema_unused_indexes`.

---

## Connection pooling — one pool per process, sized correctly

Creating a connection is expensive (TLS, auth, session setup). Reuse them via a pool.

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    settings.database_url,
    pool_size=10,            # baseline persistent connections
    max_overflow=20,         # extra connections when burst-busy (total = 30)
    pool_pre_ping=True,      # validates connection before checkout
    pool_recycle=3600,       # recycle after 1h to avoid stale TLS / DB-side timeouts
)
```

### Sizing

Approximate formula: `pool_size = max_concurrent_requests`. Going higher doesn't help — the DB itself has a connection cap (Postgres default: 100 total) and each connection costs ~10MB on the server.

For an async app with `httpx`-style fan-out, requests interleave at the await boundaries, so `pool_size` ~= `worker_count × concurrency_per_worker` understates needs. Measure: watch pool stats (`engine.pool.status()`).

**Per process, not per request.** The engine is constructed once at startup, owned by the app, shared across all requests. Never `create_engine` inside a request handler.

```python
# main.py
engine = create_engine(...)
SessionLocal = sessionmaker(engine, expire_on_commit=False)

# routes.py
@app.get("/orders/{id}")
def get_order(id: int) -> Order:
    with SessionLocal() as session:
        return session.get(Order, id) or raise_404()
```

### Async pooling

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine(settings.database_url, pool_size=10, max_overflow=20)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_order(id: int) -> Order:
    async with AsyncSessionLocal() as session:
        return await session.get(Order, id)
```

Use `asyncpg` (Postgres) or aiosqlite under the hood. Async DB drivers are mature; mixing sync DB with async server is the pain (see `13-concurrency.md`).

---

## Transactions — explicit boundaries

Every unit of work that touches multiple rows or multiple tables is one transaction. Commit on success, rollback on any exception.

```python
# BAD — implicit autocommit; partial failures leave inconsistent state
session.add(order)
session.commit()
charge_card(order.total)       # if this fails, the order exists but isn't paid
session.add(payment)
session.commit()

# GOOD — one transaction
with SessionLocal.begin() as session:    # commits on exit; rollbacks on exception
    session.add(order)
    payment = charge_card(order.total)
    session.add(payment)
```

Or with explicit control:

```python
with SessionLocal() as session:
    try:
        session.add(order)
        payment = charge_card(order.total)
        session.add(payment)
        session.commit()
    except Exception:
        session.rollback()
        raise
```

For multi-repository writes, the **Unit of Work** pattern (see `05-design-patterns.md`) makes the transaction boundary explicit at the use-case level.

### Side effects don't belong inside the transaction

Sending email, making HTTP calls, publishing events — these can't be rolled back if the transaction fails. Either:

1. **Commit first, then side-effect.** Risk: side-effect fails after commit; eventual inconsistency.
2. **Outbox pattern.** Insert the event into an `outbox` table inside the transaction. A separate worker reads `outbox` and dispatches. Atomic with the write; eventually consistent on dispatch.

```python
with SessionLocal.begin() as session:
    session.add(order)
    session.add(OutboxEvent(type="order.created", payload=order.to_dict()))
# Worker process polls outbox, publishes to message bus, marks rows sent.
```

---

## Schema design

### `NOT NULL` by default

Nullable columns are a permission slip for inconsistent data. Default to `NOT NULL`. Make a column nullable only when "the value genuinely doesn't exist yet" is a real state.

```python
class User(Base):
    id:          Mapped[int]      = mapped_column(primary_key=True)
    email:       Mapped[str]      = mapped_column(unique=True)              # NOT NULL
    name:        Mapped[str]                                                # NOT NULL
    deleted_at:  Mapped[datetime | None]                                    # nullable: legit "not yet deleted"
```

### Foreign keys, always

Every column that references another table is a foreign key with a constraint. Orphan rows from missing FK constraints are a debugging nightmare.

```python
order_id: Mapped[int] = mapped_column(ForeignKey("orders.id", ondelete="CASCADE"))
```

Pick `ondelete` deliberately: `CASCADE` (delete me with parent), `SET NULL` (parent gone, my reference becomes null), `RESTRICT` (refuse to delete parent if children exist). The default — `NO ACTION` — usually means "RESTRICT but deferrable", which is rarely what you want.

### `UUID` vs `bigint` IDs

- **`bigint`** — smaller, faster, easier to debug. Default for most internal databases.
- **`UUID`** — when IDs are exposed externally, generated by clients, or you want to merge data from multiple databases without collisions. Use `uuid7` (time-ordered) — `uuid4` is random and destroys index locality.

### Constraints over application checks

Database-level constraints survive bugs in the app. Use them.

```python
__table_args__ = (
    CheckConstraint("amount > 0", name="amount_positive"),
    CheckConstraint("status IN ('pending', 'paid', 'refunded')", name="status_valid"),
    UniqueConstraint("customer_id", "external_ref", name="uq_customer_external_ref"),
)
```

App-level validation (Pydantic) is the first line. DB constraints are the safety net.

### Enums

```python
class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"
    REFUNDED = "refunded"

status: Mapped[OrderStatus] = mapped_column(Enum(OrderStatus), index=True)
```

SQLAlchemy maps `Enum` to a native DB enum (Postgres) or a constrained `VARCHAR` (SQLite/MySQL). Either way you get DB-level validation.

---

## Migrations — Alembic, never destructive without care

Schema lives in code. Migrations are versioned and reversible. `alembic` is the default.

```bash
alembic init alembic
alembic revision --autogenerate -m "add idempotency_key to orders"
alembic upgrade head
alembic downgrade -1
```

### Migration rules

- **Never edit a migration that's been deployed.** Add a new one.
- **Migrations run before the new code starts.** Old code must work against the new schema (one-deploy-at-a-time apps don't care; rolling deploys must).
- **Reversible where possible.** A `down_revision` lets you back out.
- **Data migrations separately from schema migrations.** Schema is fast (`ALTER TABLE`); data migrations on a large table need batching and might not be fully reversible.
- **Backfill in a separate deploy.** Schema migration → backfill migration → code uses new column. Trying to do all three at once causes downtime.

### Destructive operations need ceremony

```python
# DROP TABLE / DROP COLUMN / ALTER COLUMN TYPE — never as a "looks unused, clean it up"
# Steps:
# 1. Stop writing to it (deploy code that ignores the column).
# 2. Wait long enough that no in-flight requests reference it.
# 3. Confirm via DB stats that no queries touch it.
# 4. Then drop in a separate migration.
```

The safest move with a "probably unused" column is to leave it. Disk is cheap; recovering deleted data isn't.

### Backups before risky migrations

For anything that rewrites a table, takes a lock, or alters a primary key — snapshot first. Postgres: `pg_dump`. RDS: snapshot the instance.

---

## Querying — select only what you need

```python
# BAD — loads every column, ships the wide row across the network
users = session.scalars(select(User)).all()
ids = [u.id for u in users]                    # only needed id

# GOOD — load just the columns
ids = session.scalars(select(User.id)).all()
```

For ORM queries that *only* need to enumerate, narrow the projection. For huge tables, this can be the difference between 100ms and 10s.

`limit()` everything that doesn't have a hard upper bound on row count. An unbounded `SELECT * FROM logs` will OOM the day the table grows.

---

## Bulk operations — never one row at a time

```python
# BAD — N round trips
for user in new_users:
    session.add(user)
    session.commit()

# BETTER — one transaction, N inserts
session.add_all(new_users)
session.commit()

# BEST — one INSERT with N rows
session.execute(insert(User), [u.__dict__ for u in new_users])
session.commit()
```

For tens of thousands of rows: `COPY` (Postgres) or `LOAD DATA INFILE` (MySQL). Orders of magnitude faster than even bulk inserts.

For updates: a single `UPDATE ... WHERE id IN (...)` beats N `UPDATE` statements. SQLAlchemy: `session.execute(update(User).where(User.id.in_(ids)).values(...))`.

---

## Pagination

See `12-api-design.md` for cursor pagination at the API layer. At the DB layer:

```python
# BAD — offset/limit at scale is slow (DB scans + discards N rows)
session.scalars(select(Order).order_by(Order.id).offset(10_000).limit(20)).all()

# GOOD — keyset pagination
session.scalars(
    select(Order)
    .where(Order.id > last_seen_id)
    .order_by(Order.id)
    .limit(20)
).all()
```

For small offsets (page 1–5), offset is fine. For deep pagination, keyset (cursor) is required.

---

## Raw SQL — when to reach for it

ORM is great for CRUD and most queries. Reach for raw SQL when:

- Window functions, CTEs, recursive queries.
- Bulk ops that need `INSERT ... ON CONFLICT DO UPDATE` (upserts).
- Complex aggregations that the ORM expresses awkwardly.

Use **parameterized** raw SQL via `text()`:

```python
from sqlalchemy import text

result = session.execute(
    text("""
        INSERT INTO users (email, name) VALUES (:email, :name)
        ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name
        RETURNING id
    """),
    {"email": user.email, "name": user.name},
).scalar_one()
```

Never f-string into `text()`. See `15-security.md`.

---

## Repository pattern — wrap the ORM at the boundary

Business code shouldn't import `select` and call `session.execute`. The Repository pattern (see `05-design-patterns.md`) keeps the ORM behind an interface.

```python
class OrderRepository(Protocol):
    def get(self, order_id: int) -> Order | None: ...
    def list_by_customer(self, customer_id: int) -> list[Order]: ...
    def add(self, order: Order) -> None: ...

class SQLAlchemyOrderRepository:
    def __init__(self, session: Session) -> None:
        self.session = session

    def get(self, order_id: int) -> Order | None:
        return self.session.get(Order, order_id)

    def list_by_customer(self, customer_id: int) -> list[Order]:
        return list(self.session.scalars(
            select(Order)
            .where(Order.customer_id == customer_id)
            .options(selectinload(Order.line_items))
        ))

    def add(self, order: Order) -> None:
        self.session.add(order)
```

Now services depend on `OrderRepository`, not SQLAlchemy. Tests substitute `InMemoryOrderRepository`. Swapping ORMs is a one-week project, not a one-year one.

---

## Health checks

Every app exposes `/healthz` (liveness) and `/readyz` (readiness):

- **Liveness:** "am I running?" — process is up. Return 200 unconditionally.
- **Readiness:** "can I serve traffic?" — DB connection works, dependencies reachable.

```python
@app.get("/readyz")
async def readiness() -> dict:
    try:
        await session.execute(text("SELECT 1"))
    except Exception:
        raise HTTPException(503)
    return {"status": "ready"}
```

Don't fold expensive checks into the readiness probe — Kubernetes hits it every few seconds.

---

## Long-running queries — bounded timeout

```python
# Postgres: set statement_timeout per session or per query
session.execute(text("SET LOCAL statement_timeout = '5s'"))

# Or at engine level: in connect_args
engine = create_engine(url, connect_args={"options": "-c statement_timeout=5000"})
```

A query that should take 50ms going to 50s is the moment you want it killed, not retried. Set bounds on every query path; investigate the slow query later.

---

## Anti-patterns

| Symptom | Fix |
|---|---|
| Loop over results, then access `obj.related.field` (lazy load each) | `selectinload` / `joinedload` |
| 1000 inserts in a `for` loop with commits | `session.add_all(...)` or bulk `insert()` |
| `offset=10_000` on a 1M-row table | Keyset/cursor pagination |
| No index on FK columns | `index=True` on every `ForeignKey` |
| Nullable columns by default | `NOT NULL` unless null is a real state |
| No FK constraints | Always declare them; pick `ondelete` deliberately |
| App-only validation, no DB constraints | DB `CheckConstraint` / `UniqueConstraint` as safety net |
| `pool_size=1` causing thread blocking | Size to expected concurrency; `pool_pre_ping=True` |
| `create_engine` per request | One engine per process, at startup |
| Sending email mid-transaction | Outbox pattern; or commit first then dispatch |
| `EXPLAIN` only when something's slow | Run on new query templates as part of development |
| Migration that drops a column "we don't think anyone uses" | Multi-deploy ceremony or just leave it |
| `for u in users: session.execute(update(User)...)` | Single `update().where(id.in_(ids))` |
| `f"SELECT ... {value}"` in `text(...)` | Parameterize: `text("... :v")` with `{"v": value}` |
| ORM imports leaking into business logic | Repository interface |
| Unbounded `SELECT *` from a growing table | Always `LIMIT`; project specific columns |
| No `/readyz` health check | Liveness + readiness probes |
| Query that "sometimes takes a minute" | `statement_timeout`; never a runaway |

---

## The one-screen summary

- **Eager-load relationships.** N+1 is the most common bug; `selectinload` / `joinedload` is the fix.
- **Index every FK, every WHERE column.** Composite where order matters.
- **One pool per process.** Sized to concurrency. `pool_pre_ping=True`.
- **Explicit transactions.** Commit-on-success, rollback-on-exception. Side effects via outbox.
- **`NOT NULL` by default.** FK constraints always. DB-level `CheckConstraint` as the safety net.
- **Alembic migrations, additive first.** Destructive ops are a multi-step deploy.
- **Bulk operations** for >1 row. Keyset pagination for deep paging.
- **Repository at the boundary.** Business logic doesn't know about `select`.
- **Statement timeouts** so a slow query never becomes a process hang.
