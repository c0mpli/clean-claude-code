# Logging

Logs are how you debug production. `print()` is for scripts and `__str__` demos; everything else uses the `logging` module.

---

## The rules in one screen

1. **Use stdlib `logging`** (or `structlog` for new projects that need structured output).
2. **One logger per module:** `log = logging.getLogger(__name__)` at the top of each file. Never `logging.info(...)` (that hits the root logger).
3. **Configure once, at startup.** Modules don't configure logging; the app entry point does.
4. **Levels mean something** — DEBUG / INFO / WARNING / ERROR / CRITICAL. Use them per the semantics below.
5. **Structured context via `extra={}`**, never f-strings concatenated into the message.
6. **Never log secrets, PII, or tokens.** This is what `SecretStr` is for.
7. **Don't log and re-raise.** Pick one. Log if you handle; raise if you don't.
8. **In production: JSON logs.** Indexable, queryable, parseable by the log pipeline.

---

## The logger-per-module pattern

```python
# billing/invoice.py
import logging

log = logging.getLogger(__name__)            # → "billing.invoice"

def send_invoice(invoice_id: str) -> None:
    log.info("sending invoice", extra={"invoice_id": invoice_id})
    ...
```

Why the module-named logger:
- Filterable. `logging.getLogger("billing").setLevel(DEBUG)` flips on debug for one subsystem.
- Searchable. Every log line carries its origin module.
- No collisions. Tests for `billing.invoice` can `caplog.set_level(DEBUG, logger="billing.invoice")`.

**Never** call the bare `logging.info(...)` — it goes to the root logger and you lose the source.

---

## Levels — what they mean

| Level | Meaning | Example |
|---|---|---|
| **DEBUG** | Verbose, dev-only. Off in production. | `log.debug("cache key=%s hit", key)` |
| **INFO** | Notable events in the normal flow. | `log.info("user signed up", extra={"user_id": uid})` |
| **WARNING** | Something unexpected, but the request still succeeded. | `log.warning("retrying after timeout", extra={"attempt": 3})` |
| **ERROR** | A request / job failed. Operator should look. | `log.error("payment failed", extra={"order_id": oid})` |
| **CRITICAL** | The system is unusable. Page someone. | `log.critical("database unreachable")` |

**Heuristics:**
- Most production traffic is INFO + ERROR. WARNING means "we recovered but you should know." CRITICAL means a pager fires.
- If your app emits a hundred WARNINGs per minute in steady state, they aren't warnings — they're INFO. Re-classify.
- ERROR without traceback is rare. Use `log.exception(...)` inside an `except` block — it includes the traceback automatically.

```python
try:
    process(order)
except PaymentDeclined as e:
    log.warning("payment declined", extra={"order_id": order.id, "reason": e.reason})
    return Response(402)
except Exception:
    log.exception("unexpected error processing order", extra={"order_id": order.id})
    raise
```

---

## Structured context, not f-strings

The message is the **template**. The variable data is in `extra={}`. This is what makes logs queryable.

```python
# BAD — values baked into the message; can't index "user_id" across logs
log.info(f"user {user.id} signed up with email {user.email}")
log.error(f"failed to charge {amount} to order {order.id}: {e}")

# GOOD — message is constant, fields are queryable
log.info("user signed up", extra={"user_id": user.id, "email": user.email})
log.error("charge failed", extra={"order_id": order.id, "amount": str(amount), "error": str(e)})
```

In your log pipeline (Datadog, Loki, CloudWatch, ELK), you can then query `user_id:"u_123"` directly. With f-string concatenation, you're greping over plain text.

**To use `extra` with stdlib `logging`, the formatter must include the fields.** Either use a JSON formatter (recommended for prod, see below) or write a custom one. The `extra` dict is attached to the `LogRecord` regardless; rendering it is the formatter's job.

---

## Configure once, at startup

```python
# main.py
import logging.config

def configure_logging(level: str = "INFO", json_logs: bool = False) -> None:
    logging.config.dictConfig({
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "json": {
                "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
                "format": "%(asctime)s %(name)s %(levelname)s %(message)s",
            },
            "plain": {
                "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
            },
        },
        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "formatter": "json" if json_logs else "plain",
            },
        },
        "root": {"handlers": ["console"], "level": level},
        "loggers": {
            # Quiet noisy libraries
            "urllib3": {"level": "WARNING"},
            "botocore": {"level": "WARNING"},
        },
    })


# Entry point
def main() -> None:
    settings = Settings()
    configure_logging(level=settings.log_level, json_logs=settings.environment == "prod")
    run(settings)
```

What this gives you:
- **One place** that knows about log formatters / handlers / levels.
- Plain text in local dev, JSON in production.
- Library noise (urllib3, boto, sqlalchemy) explicitly bounded.

---

## JSON logs in production

Local dev wants human-readable lines. Production wants machine-readable JSON.

```python
# Install: pip install python-json-logger
from pythonjsonlogger import jsonlogger

handler = logging.StreamHandler()
handler.setFormatter(jsonlogger.JsonFormatter(
    "%(asctime)s %(name)s %(levelname)s %(message)s",
    rename_fields={"asctime": "timestamp", "levelname": "level"},
))
```

Every log line becomes:

```json
{"timestamp": "2026-05-18T12:34:56", "level": "INFO", "name": "billing.invoice",
 "message": "sending invoice", "invoice_id": "inv_123", "user_id": "u_456"}
```

Indexable on every field. Filterable in Kibana / Datadog / Loki without parsing.

---

## Correlation IDs — for tracing requests across services

A single user request hits multiple services / queues / workers. You want every log line for that request to carry the same ID.

```python
# middleware.py
import contextvars
import uuid

request_id_var: contextvars.ContextVar[str] = contextvars.ContextVar("request_id", default="-")

class RequestIDMiddleware:
    async def __call__(self, request, call_next):
        req_id = request.headers.get("x-request-id") or str(uuid.uuid4())
        token = request_id_var.set(req_id)
        try:
            response = await call_next(request)
            response.headers["x-request-id"] = req_id
            return response
        finally:
            request_id_var.reset(token)


# logging filter — injects request_id into every record
class RequestIDFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id_var.get()
        return True


# Then attach the filter in dictConfig and reference it in the formatter:
# "format": "%(asctime)s %(request_id)s %(name)s %(message)s"
```

Every log line in that request now carries `request_id=<uuid>`. Cross-service tracing on top of this uses OpenTelemetry — same idea, more fields.

---

## Never log secrets, PII, or tokens

```python
# BAD
log.info("login", extra={"email": user.email, "password": password})    # password
log.info("payment", extra={"card_number": card.number})                  # PCI
log.debug("request", extra={"headers": dict(request.headers)})           # Authorization header

# GOOD
log.info("login", extra={"user_id": user.id})
log.info("payment", extra={"order_id": order.id, "card_last4": card.number[-4:]})
log.debug("request", extra={"headers": {k: v for k, v in request.headers.items()
                                         if k.lower() not in SENSITIVE_HEADERS}})
```

Define `SENSITIVE_HEADERS` and `SENSITIVE_FIELDS` as constants and apply them in a logging filter. Don't trust ad-hoc redaction at the call site — someone will forget.

```python
SENSITIVE_HEADERS = {"authorization", "cookie", "x-api-key"}
SENSITIVE_FIELDS  = {"password", "secret", "token", "api_key", "ssn", "credit_card"}

class RedactSensitiveFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        for field in SENSITIVE_FIELDS:
            if hasattr(record, field):
                setattr(record, field, "***")
        return True
```

`SecretStr` (from `04-error-handling.md` / `09-configuration.md`) helps here: `SecretStr.__repr__` already returns `SecretStr('**********')`, so logging a settings object doesn't leak.

---

## Don't log-and-raise

```python
# BAD — duplicate noise; whoever catches it logs again
try:
    charge(order)
except PaymentError as e:
    log.error("charge failed: %s", e)
    raise

# GOOD — let the exception propagate; log at the boundary that handles it
try:
    charge(order)
except PaymentError as e:
    log.warning("charge declined", extra={"order_id": order.id, "reason": e.reason})
    return Response(402, body={"error": "payment_declined"})
```

Rule: **the boundary that handles the error logs it.** Everywhere else, let exceptions propagate. The top of the call stack — the HTTP handler, the background-job runner, the CLI entry point — has the most context to decide log level and structure.

Use `log.exception(...)` inside an `except` block only when you're catching an unexpected exception **at the boundary**. It includes the traceback automatically.

---

## `caplog` in tests

`pytest`'s built-in `caplog` fixture captures log records.

```python
def test_payment_declined_logs_warning(caplog):
    with caplog.at_level(logging.WARNING, logger="billing.payment"):
        charge_or_decline(order=declined_order)

    [record] = [r for r in caplog.records if r.message == "charge declined"]
    assert record.levelname == "WARNING"
    assert record.order_id == declined_order.id
    assert record.reason == "insufficient_funds"
```

Assert on **records and their fields**, not on the formatted string. The format can change; the record's structure shouldn't.

---

## `structlog` — when stdlib gets clumsy

For new projects with rich structured logging needs, `structlog` is cleaner than stdlib + `extra`:

```python
import structlog

log = structlog.get_logger()

log.info("user signed up", user_id=user.id, email=user.email, plan="pro")
log.warning("retrying after timeout", attempt=3, max_attempts=5, endpoint="/api/v1/charge")
```

You get keyword arguments instead of `extra={}`, automatic structured output, and seamless integration with stdlib (handlers, filters). Configure once at startup; from then on, every log line is JSON with consistent fields.

Stick with stdlib `logging` if you have an existing project; `structlog` for greenfield.

---

## Production observability stack

Logs are one of three pillars. The full stack:

- **Logs** — discrete events. Stdlib `logging` → JSON → log pipeline (Loki, Datadog, ELK).
- **Metrics** — numeric time-series (latency, error rate, throughput). Prometheus / Datadog.
- **Traces** — request paths across services. OpenTelemetry → Jaeger / Tempo / Honeycomb.

Don't reach for logs to count things. Counters are metrics. Don't grep logs to follow a request — that's what traces are for. Use the right pillar for the question.

---

## Anti-patterns

### `print()` for production output

```python
# BAD
print(f"user {user.id} signed up")

# GOOD
log.info("user signed up", extra={"user_id": user.id})
```

`print` is fine in scripts, CLIs talking to humans, and `__str__`/`__repr__`-style demos. **Not** in services, background jobs, or library code.

### Logging inside tight loops

```python
# BAD
for item in millions_of_items:
    log.debug("processing", extra={"item_id": item.id})    # 10M log lines

# GOOD
log.info("processing batch", extra={"count": len(items)})
for item in items:
    ...
log.info("batch complete", extra={"count": len(items), "duration_s": elapsed})
```

Log boundaries (batch start/end), not every iteration. If you need per-item debug, sample it (`if i % 1000 == 0: log.debug(...)`).

### Logging the same event at multiple levels

```python
# BAD — same event logged in two layers, two formats
def handler():
    log.info("processing payment")
    try:
        service.process(payment)         # service also logs "processing payment"
    except Exception:
        log.exception("payment failed")

```

Decide *one* layer that owns the event. Usually the lowest-level function that can name the event meaningfully. Higher layers log only their own concerns (e.g., HTTP status, queue acknowledgement).

### `log.error(str(e))` — losing the traceback

```python
# BAD
except Exception as e:
    log.error(str(e))                # no traceback, no exception type

# GOOD
except Exception:
    log.exception("operation failed", extra={"operation": "charge_card"})
```

`log.exception` is `log.error` plus the traceback. Always use it inside an `except` block.

### Configuring logging in library code

```python
# BAD — your library shouldn't add handlers
# my_library/__init__.py
logging.basicConfig(level=logging.INFO)

# GOOD — your library only gets a logger; the app configures handlers
log = logging.getLogger(__name__)
```

Libraries acquire loggers and emit records. Applications configure handlers and levels. Mixing the two means importing your library reconfigures the host app's logging.

---

## Quick reference

| Symptom | Fix |
|---|---|
| `print()` in service code | `log.info(...)` |
| `logging.info(...)` (root logger) | `log = logging.getLogger(__name__)` per module |
| `log.info(f"user {uid} did X")` | `log.info("user did X", extra={"user_id": uid})` |
| `log.error(str(e))` inside `except` | `log.exception("...")` |
| Logging and re-raising | Log at the boundary that handles; let it propagate elsewhere |
| Library calls `basicConfig` | Library only gets a logger; app configures |
| Secret / token in log | Redact via filter; `SecretStr` for config |
| Plain-text logs in prod | JSON formatter + log pipeline |
| Test asserts on formatted string | Assert on `caplog.records` fields |
| Following one request across logs | Correlation ID via `ContextVar` + logging filter |
| Logging every item in a 10M loop | Log batch boundaries; sample inside |
