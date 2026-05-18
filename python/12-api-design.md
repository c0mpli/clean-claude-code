# API Design (HTTP / REST)

How to shape HTTP APIs so consumers can read them once and use them forever. Examples use FastAPI; the rules are framework-independent.

---

## URLs name resources, not actions

URLs are nouns. Verbs are HTTP methods. Resources are plural.

```
# BAD                              # GOOD
POST /createUser                   POST /users
GET  /getUserById/123              GET  /users/123
POST /user/123/activate            POST /users/123/activations    (creates an activation)
GET  /listOrdersForUser/123        GET  /users/123/orders
POST /deleteOrder/456              DELETE /orders/456
```

The HTTP verb says what to do. The path says to what. If you find yourself reaching for a verb in the URL (`/activate`, `/cancel`, `/process`), model the action as a sub-resource (`POST /orders/{id}/cancellations`).

---

## HTTP methods — semantics, not vibes

| Method | Safe? | Idempotent? | Body? | Use for |
|---|---|---|---|---|
| `GET` | yes | yes | no | Read a resource or list |
| `HEAD` | yes | yes | no | Headers only — existence checks, ETag fetching |
| `POST` | no | **no** | yes | Create, or non-idempotent action |
| `PUT` | no | yes | yes | Full replace of a resource |
| `PATCH` | no | yes (should be) | yes | Partial update |
| `DELETE` | no | yes | no | Remove a resource |
| `OPTIONS` | yes | yes | no | CORS preflight, capability discovery |

**Safe:** doesn't change server state. Crawlers can call it freely.
**Idempotent:** N identical calls have the same effect as 1.

Get these wrong and proxies, caches, and clients will break in subtle ways. Never put a state change inside a `GET`. Make `DELETE` and `PUT` idempotent so retries are safe.

---

## Status codes — the small useful set

| Code | Meaning | Use when |
|---|---|---|
| `200 OK` | Generic success | Successful GET, PATCH, or POST that doesn't create |
| `201 Created` | Resource created | Successful POST that created one; include `Location` header |
| `202 Accepted` | Request accepted, not yet processed | Async / queued work |
| `204 No Content` | Success, nothing to return | Successful DELETE |
| `400 Bad Request` | Client sent something malformed | Validation failure |
| `401 Unauthorized` | No credentials, or invalid | Missing/expired token |
| `403 Forbidden` | Authenticated but not allowed | Permission denied |
| `404 Not Found` | Resource doesn't exist | Bad ID, or auth-aware "not found" |
| `409 Conflict` | State conflict | Optimistic-lock failure, duplicate email |
| `410 Gone` | Permanently removed | Soft-deleted resources you won't restore |
| `422 Unprocessable Entity` | Syntactically valid, semantically wrong | Sometimes preferred over 400 for body validation |
| `429 Too Many Requests` | Rate-limited | Include `Retry-After` header |
| `500 Internal Server Error` | Unexpected | Bug. Don't leak the traceback. |
| `502/503/504` | Upstream / overload / timeout | Infrastructure failures |

**Heuristics:**
- 4xx = client's fault. The client can fix the request and retry.
- 5xx = server's fault. The client should retry with backoff or surface the error.
- Never return 200 with `{"error": "..."}` in the body. The status code IS the error.
- `401` vs `403`: 401 means "I don't know who you are." 403 means "I know who you are; you can't do this."

---

## Response shape — pick one envelope, stick with it

Two reasonable defaults. **Pick one for the whole API.**

### Option A: naked resource

```json
GET /users/123 → 200
{
  "id": 123,
  "email": "a@b.c",
  "created_at": "2026-05-18T12:00:00Z"
}

GET /users → 200
{
  "data": [{"id": 1, ...}, {"id": 2, ...}],
  "next_cursor": "eyJpZCI6Mn0="
}
```

Single resources are naked; collections need pagination metadata so they live in an envelope.

### Option B: every response in an envelope

```json
GET /users/123 → 200
{
  "data": {"id": 123, "email": "a@b.c", ...},
  "meta": {"request_id": "req_abc"}
}
```

Consistent shape, easier for generic clients. Slightly more typing.

**The rule is consistency.** Don't ship `/users/123` as naked and `/orders/456` enveloped.

---

## Error responses — use a standard shape

[RFC 7807 `application/problem+json`](https://datatracker.ietf.org/doc/html/rfc7807) is the standard. Use it.

```json
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://errors.example.com/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "email is not a valid address",
  "instance": "/users",
  "errors": [
    {"field": "email", "message": "not a valid email address"},
    {"field": "age",   "message": "must be >= 13"}
  ],
  "request_id": "req_abc123"
}
```

Required: `type`, `title`, `status`. Add `detail`, `instance`, `errors`, `request_id` as useful.

Why this matters: every client builds the same error-display code if every API returns the same shape. The alternative — every endpoint inventing its own error JSON — is what makes API integration painful.

```python
# FastAPI
from fastapi import HTTPException
from fastapi.responses import JSONResponse

@app.exception_handler(ValidationError)
async def validation_handler(request, exc: ValidationError) -> JSONResponse:
    return JSONResponse(
        status_code=422,
        media_type="application/problem+json",
        content={
            "type": "https://errors.example.com/validation",
            "title": "Validation failed",
            "status": 422,
            "errors": [{"field": ".".join(map(str, e["loc"])), "message": e["msg"]} for e in exc.errors()],
            "request_id": request.state.request_id,
        },
    )
```

**Never** return a 500 with a stack trace in the body. Log the trace server-side; return a generic problem+json with `request_id` so support can correlate.

---

## Pagination — cursor over offset

For data that changes, `offset` / `limit` is wrong: insertions and deletions during pagination produce duplicates and skips.

```
# BAD — offset pagination on a changing list
GET /orders?offset=0&limit=20
GET /orders?offset=20&limit=20   # if 3 orders were inserted, you skip 3
```

```
# GOOD — cursor pagination
GET /orders?limit=20
→ {
    "data": [...],
    "next_cursor": "eyJpZCI6MTIzNDV9"
  }

GET /orders?limit=20&cursor=eyJpZCI6MTIzNDV9
→ {
    "data": [...],
    "next_cursor": null               # last page when null/absent
  }
```

The cursor is opaque to the client — base64 of `{"id": <last_id>}` or whatever the server needs to resume. Clients pass it back verbatim.

**Bounds:** always cap `limit` (e.g., max 100). A client asking for `limit=1000000` should get 100, not crash you.

**Sorting:** if the API supports sort, the cursor must encode the sort field too. `cursor` paired with `?sort=created_at:desc` resumes from the last `(created_at, id)` pair.

---

## Filtering and sorting — query params

```
GET /orders?status=paid                          # single filter
GET /orders?status=paid,refunded                 # multiple values (comma)
GET /orders?status=paid&customer_id=123          # combined filters (AND)
GET /orders?created_after=2026-01-01             # range
GET /orders?sort=-created_at,total               # negative prefix for desc
GET /orders?fields=id,status,total               # sparse fieldsets
GET /orders?include=customer,line_items          # related-resource inclusion
```

Don't invent a query DSL in the URL. If filtering gets richer than "field = value AND field IN values AND field BETWEEN x AND y", build a dedicated `POST /orders/search` endpoint that takes a JSON body.

---

## Versioning — URL prefix is the simplest

```
/v1/users
/v2/users
```

Pros: visible at a glance, trivial to route, works with every HTTP tool.

Header-based versioning (`Accept: application/vnd.example.v2+json`) is more "correct" by REST purism and a nightmare in practice. Engineers debugging in curl forget the header; CDN caches don't key on it by default.

**When to bump the version:** breaking change to the response shape, removed endpoints, semantic change to existing fields. Adding optional fields and new endpoints is **not** breaking — don't bump for those.

Run two versions in parallel during deprecation. Add a `Sunset` header to the old one (RFC 8594) with the removal date.

---

## Idempotency keys — for non-idempotent operations

`POST /payments` is not idempotent. A network blip + client retry creates two charges. The fix is an idempotency key.

```
POST /payments
Idempotency-Key: req_abc123
Content-Type: application/json

{"amount": "10.00", "currency": "USD", "source": "tok_xyz"}
```

Server stores `(idempotency_key, request_hash, response)` for ~24h. A duplicate request with the same key:
- Same body → return the cached response (the original outcome).
- Different body → 422 with "Idempotency-Key reused for different request".

```python
async def post_payment(req: CreatePaymentRequest, idempotency_key: str = Header(...)) -> Payment:
    if cached := await idempotency_store.get(idempotency_key):
        if cached.request_hash != req.content_hash():
            raise HTTPException(422, "Idempotency-Key reused with different body")
        return cached.response

    payment = await create_payment(req)
    await idempotency_store.put(idempotency_key, req.content_hash(), payment, ttl_s=86400)
    return payment
```

Every state-changing endpoint that clients retry should accept an `Idempotency-Key`. Stripe popularized this pattern; copy it.

---

## Rate limiting — tell the client what's happening

Return rate-limit info in headers so clients can self-regulate without guessing.

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1715856000        # Unix timestamp when window resets
```

When throttled:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60                       # seconds (or HTTP-date)
Content-Type: application/problem+json

{
  "type": "https://errors.example.com/rate-limited",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "1000 requests per hour exceeded"
}
```

`Retry-After` is the contract. Honest clients respect it; dishonest ones get blocked.

---

## Caching — `ETag` + conditional requests

For resources that change infrequently, support conditional GETs:

```
GET /users/123 → 200
ETag: "abc123"
Cache-Control: private, max-age=60

{"id": 123, ...}

# Later, client revalidates:
GET /users/123
If-None-Match: "abc123"

→ 304 Not Modified                    # body omitted; client uses cached copy
```

Saves bandwidth and CPU. The `ETag` is usually a hash of the response body or a version counter on the resource.

---

## Auth — Bearer tokens in the standard header

```
GET /users/me
Authorization: Bearer eyJhbGc...
```

Don't reinvent — use the standard `Authorization: Bearer <token>` header. Cookies for browser apps (with `Secure`, `HttpOnly`, `SameSite=Lax`), Bearer tokens for everything else.

**Never put secrets in URLs.** `?api_key=xyz` ends up in server logs, browser history, and HTTP referers. Always headers (or body for non-GET).

---

## Field naming

- **`snake_case` for JSON keys** in Python ecosystems. `camelCase` is fine if your house style is JS-first; pick one and stay consistent across every endpoint.
- **`id` is a string** unless the API is internal-only and you control all clients. Numeric IDs > `2^53` lose precision in JavaScript.
- **Timestamps are ISO 8601 strings with `Z` or `+00:00`**. Never Unix epoch seconds (loses precision and timezone info). See `11-dates-money.md`.
- **Money is `{"amount": "10.00", "currency": "USD"}`**, never a bare number. JSON has one number type; string-encoded decimals preserve precision.
- **Booleans are `true`/`false`**, not `0`/`1` or `"yes"`/`"no"`.

---

## Request/response bodies — validate at the boundary

This is the rule from `04-error-handling.md` applied to HTTP. Inputs are Pydantic models; outputs are Pydantic models. Internal code sees only validated types.

```python
class CreateOrderRequest(BaseModel):
    customer_id: int
    items: list[OrderItemRequest] = Field(min_length=1, max_length=100)
    idempotency_key: str | None = Field(default=None, max_length=128)

class OrderResponse(BaseModel):
    id: int
    customer_id: int
    status: OrderStatus
    total: Money
    created_at: AwareDatetime

@app.post("/orders", response_model=OrderResponse, status_code=201)
async def create_order(req: CreateOrderRequest) -> OrderResponse:
    order = await order_service.create(...)
    return OrderResponse.from_model(order)
```

`response_model=OrderResponse` is the contract. Anything the service returns that isn't in `OrderResponse` is stripped. Anything that *should* be there but isn't is an error. Documentation generation, schema validation, and OpenAPI all come for free.

---

## OpenAPI — generated from code

Hand-written API docs rot the moment the code changes. With FastAPI / Pydantic, OpenAPI is generated from your model definitions and route signatures. You get `/docs` (Swagger UI) and `/redoc` for free, and they're always accurate.

```python
@app.post(
    "/orders",
    response_model=OrderResponse,
    status_code=201,
    responses={
        409: {"model": ProblemDetails, "description": "Duplicate idempotency key"},
        422: {"model": ProblemDetails, "description": "Validation error"},
    },
    tags=["orders"],
    summary="Create an order",
)
async def create_order(req: CreateOrderRequest) -> OrderResponse: ...
```

The `summary` and tags shape the rendered docs. The `responses` map documents error shapes alongside the happy path.

---

## Anti-patterns

| Symptom | Fix |
|---|---|
| `POST /createUser` | `POST /users` |
| `GET /getUserById/123` | `GET /users/123` |
| 200 OK with `{"error": "..."}` body | Use the right 4xx/5xx status code |
| 500 with stack trace in body | Log server-side; return generic problem+json |
| Custom error shape per endpoint | RFC 7807 `application/problem+json` |
| `offset=N&limit=M` on a changing list | Cursor pagination |
| Unbounded `limit` | Cap at 100 (or whatever) server-side |
| Version in `Accept` header only | `/v1/...` URL prefix |
| Breaking change without version bump | Bump the version, deprecate the old |
| `POST /payments` without idempotency key | Accept and persist `Idempotency-Key` |
| No `X-RateLimit-*` headers | Always include — clients need them to self-throttle |
| `?api_key=...` in query string | `Authorization: Bearer <token>` header |
| Unix timestamps as numbers | ISO 8601 strings |
| Money as `"amount": 10.0` | `{"amount": "10.00", "currency": "USD"}` |
| ID as 64-bit int in JSON | String, especially for cross-platform APIs |
| Hand-maintained API docs | Generate from Pydantic + FastAPI / `drf-spectacular` / etc. |

---

## Quick reference

The seven things to get right and your API will be 90% good:

1. **Resource-noun URLs**, HTTP verbs for actions.
2. **Correct status codes** — never 200 on errors.
3. **One error shape everywhere** — RFC 7807.
4. **Cursor pagination** with bounded `limit`.
5. **Idempotency keys** on every state-changing endpoint.
6. **Pydantic at the boundary** — validated requests in, validated responses out.
7. **OpenAPI generated from code** — never hand-written.
