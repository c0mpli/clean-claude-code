# Concurrency

How to pick between `async`, threads, and processes — and how to write each correctly. Most "make it concurrent" attempts pick the wrong model and end up slower than the sequential version.

---

## The decision tree

```
What kind of work?
├── I/O-bound (network, disk, database)
│   ├── New code, framework supports it     → async (asyncio)
│   └── Existing sync code with one slow call → threads (concurrent.futures)
│
├── CPU-bound (compute, image processing, ML)
│   └── → processes (multiprocessing / ProcessPoolExecutor)
│
└── Mixed: dispatch CPU work from async    → asyncio.to_thread / asyncio.loop.run_in_executor
```

**The rule of thumb:** if you can't say which axis your bottleneck is on, *measure first*. Adding the wrong concurrency model makes code slower and harder to debug.

---

## When NOT to use concurrency

Concurrency is overhead. Reach for it only when there's a measured win.

- **Sequential is fine.** A script that runs once a day and finishes in 30s doesn't need async. Don't async-ify it.
- **The hot loop is CPU-light and small.** Concurrency adds context-switch and serialization cost. For 1000 dict lookups, sequential wins.
- **The bottleneck is one slow call upstream.** Making 50 parallel requests to an API that rate-limits at 5/s makes things *slower*. Fix the bottleneck, not the call site.

---

## Async (`asyncio`) — for I/O-bound concurrency

Use async when you have many I/O calls (HTTP, database, file) and want to overlap their waiting time. **Pick a framework that's async-native** (FastAPI, Starlette, httpx, asyncpg, aiosqlite). Mixing sync and async libraries is where the pain starts.

### The basic shape

```python
import asyncio
import httpx

async def fetch_user(client: httpx.AsyncClient, user_id: int) -> User:
    response = await client.get(f"/users/{user_id}")
    response.raise_for_status()
    return User(**response.json())

async def fetch_all(user_ids: list[int]) -> list[User]:
    async with httpx.AsyncClient(base_url="https://api.example.com") as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [tg.create_task(fetch_user(client, uid)) for uid in user_ids]
    return [t.result() for t in tasks]
```

### `TaskGroup` (3.11+) over `asyncio.gather`

`TaskGroup` gives you structured concurrency: if one task raises, the others are cancelled cleanly, and the exception propagates with proper context.

```python
# OK — gather; one task's failure doesn't cancel siblings unless return_exceptions=False AND you catch it
results = await asyncio.gather(*tasks)

# BETTER — TaskGroup; failures cancel siblings, clean propagation
async with asyncio.TaskGroup() as tg:
    for x in xs:
        tg.create_task(process(x))
# All tasks done here, or one failed and the rest were cancelled and the exception raised
```

Reach for `gather` only when you specifically want fire-and-forget semantics (`return_exceptions=True` to collect errors as values).

### Bound concurrency with a semaphore

Unbounded concurrency overwhelms the server you're calling, hits rate limits, and exhausts local file handles. Always cap.

```python
async def fetch_all(urls: list[str], *, concurrency: int = 10) -> list[Response]:
    sem = asyncio.Semaphore(concurrency)
    async def fetch_bounded(url: str) -> Response:
        async with sem:
            return await client.get(url)

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch_bounded(u)) for u in urls]
    return [t.result() for t in tasks]
```

10–20 is a reasonable starting cap for HTTP calls to one host. Adjust based on the upstream's rate limit.

### Timeouts on every external call

```python
async with asyncio.timeout(5):           # 3.11+; raises TimeoutError on expiry
    response = await client.get(url)
```

Or per-client at construction (`httpx.AsyncClient(timeout=httpx.Timeout(5.0))`). A call without a timeout will hang forever — that's not concurrent, that's deadlocked.

### Never `time.sleep` in async code

```python
# BAD — blocks the event loop; all other tasks pause
async def poll():
    while True:
        check()
        time.sleep(1)                    # !!

# GOOD
async def poll():
    while True:
        check()
        await asyncio.sleep(1)
```

The whole point of async is overlap. `time.sleep` blocks the loop and serializes everything.

### Never call blocking I/O from async

Same rule generalized. If you must call a sync function that does I/O (legacy library, CPU-bound work), wrap it.

```python
import asyncio

# 3.9+
result = await asyncio.to_thread(blocking_function, arg1, arg2)

# Older / more control
loop = asyncio.get_running_loop()
result = await loop.run_in_executor(None, blocking_function, arg1, arg2)
```

`to_thread` runs in the default thread pool. For CPU-bound, use a `ProcessPoolExecutor` instead (see below).

### Queues for producer-consumer

`asyncio.Queue` for hand-off between an async producer and one or more async consumers. Backpressure for free via `maxsize`.

```python
async def producer(queue: asyncio.Queue[Job]) -> None:
    async for job in fetch_jobs():
        await queue.put(job)             # awaits if queue is full → backpressure
    await queue.put(None)                # sentinel for done

async def consumer(queue: asyncio.Queue[Job], worker_id: int) -> None:
    while True:
        job = await queue.get()
        if job is None:
            await queue.put(None)        # cascade sentinel for other consumers
            break
        try:
            await process(job)
        finally:
            queue.task_done()

async def run(workers: int = 5) -> None:
    queue: asyncio.Queue[Job | None] = asyncio.Queue(maxsize=100)
    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue))
        for w in range(workers):
            tg.create_task(consumer(queue, w))
```

`maxsize` is the backpressure knob. Without it, a fast producer + slow consumers = OOM.

### Cancellation safety

When a task is cancelled, `asyncio.CancelledError` is raised at the next `await`. Resources need cleanup.

```python
# BAD — connection leaks if cancelled mid-block
async def fetch(url: str) -> str:
    conn = await pool.acquire()
    data = await conn.get(url)
    await pool.release(conn)
    return data

# GOOD — async with handles cleanup on cancellation
async def fetch(url: str) -> str:
    async with pool.acquire() as conn:
        return await conn.get(url)
```

Always use `async with` for resources. The context manager's `__aexit__` runs on cancellation; manual `acquire/release` doesn't.

### Don't catch `CancelledError` silently

```python
# BAD
try:
    await long_op()
except Exception:                        # catches CancelledError on older Python
    log.error("oops")

# GOOD
try:
    await long_op()
except asyncio.CancelledError:
    raise                                # let cancellation propagate
except Exception as e:
    log.error("operation failed", extra={"error": str(e)})
```

In Python 3.8+, `CancelledError` inherits from `BaseException` (not `Exception`), so `except Exception` won't catch it — but the rule still holds: if you do see `CancelledError`, re-raise it. Swallowing it breaks structured concurrency.

---

## Threads — for blocking I/O when async isn't an option

When a library is sync-only and async-ifying it isn't realistic (large legacy SDK, DB driver with no async port), use threads.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch_blocking(url: str) -> str:
    return requests.get(url).text

def fetch_all(urls: list[str], *, max_workers: int = 10) -> list[str]:
    with ThreadPoolExecutor(max_workers=max_workers) as pool:
        return list(pool.map(fetch_blocking, urls))

# Or with as_completed for streaming results
def fetch_all_streaming(urls: list[str]) -> Iterator[str]:
    with ThreadPoolExecutor(max_workers=10) as pool:
        futures = {pool.submit(fetch_blocking, u): u for u in urls}
        for future in as_completed(futures):
            yield future.result()
```

**Don't share mutable state across threads.** If you must, use `threading.Lock` or — better — `queue.Queue` to hand work between threads. Locks are easy to get wrong.

### The GIL — what it actually means

The GIL serializes Python bytecode execution. **Two threads cannot run Python code simultaneously.** They can:
- Wait on I/O simultaneously (which is why threads help for I/O-bound).
- Run C extensions that release the GIL simultaneously (numpy operations, encryption, etc.).

For pure-Python CPU work, threads do not parallelize. Use processes.

Python 3.13+ has an experimental no-GIL build (PEP 703). It's not the default yet; assume the GIL exists.

---

## Processes — for CPU-bound work

When you're doing actual computation in Python (parsing, image manipulation, ML inference in pure-Python code), use processes.

```python
from concurrent.futures import ProcessPoolExecutor

def expensive(item: bytes) -> Result:
    return run_cpu_heavy_thing(item)

def process_all(items: list[bytes]) -> list[Result]:
    with ProcessPoolExecutor(max_workers=os.cpu_count()) as pool:
        return list(pool.map(expensive, items))
```

### Pickling caveat — every argument and return crosses a process boundary

Args and return values are pickled and unpickled. This means:
- **No lambdas, no nested functions, no closures** — they can't pickle.
- **No file handles, sockets, database connections** — open them inside the worker.
- **Large objects are expensive to ship.** Pass IDs / paths, not 1GB arrays. Or use `shared_memory`.

```python
# BAD — closure over `model`; will fail to pickle
def run(model: Model) -> Callable:
    def task(x: int) -> int:
        return model.predict(x)
    with ProcessPoolExecutor() as pool:
        return list(pool.map(task, range(100)))

# GOOD — module-level function; pass everything explicitly
def predict_one(model_path: str, x: int) -> int:
    model = load_model(model_path)       # loaded inside worker
    return model.predict(x)

def run(model_path: str) -> list[int]:
    with ProcessPoolExecutor() as pool:
        return list(pool.map(predict_one, [model_path] * 100, range(100)))
```

Better: load the model once per worker via the `initializer` argument:

```python
def init_worker(model_path: str) -> None:
    global _model
    _model = load_model(model_path)

def predict_one(x: int) -> int:
    return _model.predict(x)

with ProcessPoolExecutor(max_workers=8, initializer=init_worker, initargs=("model.pkl",)) as pool:
    results = list(pool.map(predict_one, items))
```

### Dispatching CPU work from async

```python
async def handle_request(image: bytes) -> Result:
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(process_pool, expensive_op, image)
```

Pass a `ProcessPoolExecutor` as the executor. The async server stays responsive; the CPU work runs in a worker process.

---

## Async testing

```python
# Install pytest-asyncio
# pyproject.toml:
# [tool.pytest.ini_options]
# asyncio_mode = "auto"

@pytest.mark.asyncio
async def test_fetch_user_returns_user(client: httpx.AsyncClient) -> None:
    user = await fetch_user(client, user_id=1)
    assert user.id == 1


@pytest_asyncio.fixture
async def client() -> AsyncIterator[httpx.AsyncClient]:
    async with httpx.AsyncClient(base_url="https://api.example.com") as c:
        yield c
```

For real test isolation against an async HTTP service, use `httpx.AsyncClient(transport=ASGITransport(app=app))` — runs requests against your app in-process without a network stack.

---

## Common anti-patterns

### `asyncio.run` inside a function called from async code

```python
# BAD — nested event loops crash with "asyncio.run() cannot be called from a running event loop"
async def outer():
    asyncio.run(inner())                 # !!

# GOOD — just await
async def outer():
    await inner()
```

`asyncio.run` is for the entry point of the program, called once.

### Manually creating tasks and forgetting them

```python
# BAD — task is garbage-collected mid-flight; "Task was destroyed but it is pending!"
async def fire():
    asyncio.create_task(do_thing())      # nobody holds a reference
    return

# GOOD — keep the reference; or use TaskGroup
async def fire():
    task = asyncio.create_task(do_thing())
    await task
```

Garbage collection of running tasks is a real bug. Either await them or hold them in a set you clean up.

### Mixing sync and async by calling `loop.run_until_complete` from sync code that's *inside* a server

```python
# BAD inside a sync framework that's running on top of an event loop
def handler():
    asyncio.get_event_loop().run_until_complete(async_thing())
```

If you're in a sync framework, stay sync. If you're in an async framework, stay async. Mixing them halfway is the path to "deadlock on the seventh request".

### Sharing an `httpx.AsyncClient` per-request

```python
# BAD — connection pool isn't reused; TLS handshake every time
async def fetch(url):
    async with httpx.AsyncClient() as client:
        return await client.get(url)

# GOOD — one client for the app's lifetime
client = httpx.AsyncClient()

@app.on_event("shutdown")
async def cleanup():
    await client.aclose()

async def fetch(url):
    return await client.get(url)
```

Reuse the client. Connection pooling is the win.

### Threads + asyncio without `run_in_executor`

A sync function with `time.sleep(10)` called from async will freeze the loop for 10 seconds. Always `asyncio.to_thread(sync_func)` or wrap with `run_in_executor`.

---

## Quick reference

| Symptom | Fix |
|---|---|
| Don't know if it's CPU- or I/O-bound | Measure first; don't add concurrency blind |
| `time.sleep(...)` in async function | `await asyncio.sleep(...)` |
| Blocking library call in async | `await asyncio.to_thread(func, ...)` |
| Unbounded `gather` over a thousand URLs | `Semaphore(N)` to cap concurrency |
| `asyncio.gather` for fail-together semantics | `asyncio.TaskGroup` (3.11+) |
| `requests` in async code | Switch to `httpx.AsyncClient` |
| No timeout on external call | `async with asyncio.timeout(N)` or client-level timeout |
| `httpx.AsyncClient` per request | One client per app, closed on shutdown |
| Threads for CPU-bound Python work | Processes (`ProcessPoolExecutor`) |
| Lambda / closure into `ProcessPoolExecutor` | Module-level function; `initializer` for per-worker setup |
| `except Exception` swallowing `CancelledError` | Catch `CancelledError` first and re-raise |
| Manual `acquire`/`release` of a connection in async | `async with` |
| `asyncio.run` called from inside another async function | Just `await` |
| Task created without being awaited or retained | `TaskGroup`, or hold the reference until it completes |

---

## The one-line summary

- **I/O-bound + greenfield → async.** TaskGroup, Semaphore, timeouts, never `time.sleep`.
- **I/O-bound + legacy sync libs → threads.** `ThreadPoolExecutor`, `queue.Queue` for hand-off.
- **CPU-bound → processes.** `ProcessPoolExecutor`, module-level worker functions, `initializer` for per-worker state.
- **In doubt → measure.** Concurrency is overhead; the wrong model is slower than sequential.
