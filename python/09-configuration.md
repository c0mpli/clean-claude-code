# Configuration

Config is data. Load it once, validate it once, freeze it, inject it. Never reach for `os.environ.get(...)` from inside business logic.

---

## The default: `pydantic-settings`

Use `pydantic-settings`'s `BaseSettings` for every app. It gives you typed config, validation, `.env` parsing, env-var loading, and secret handling in one place.

```python
from pydantic import Field, PostgresDsn, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="forbid",                  # unknown env vars are errors, not silently ignored
    )

    # Connection
    database_url: PostgresDsn
    redis_url: str = "redis://localhost:6379/0"

    # Secrets — SecretStr keeps them out of logs and repr
    secret_key: SecretStr
    stripe_api_key: SecretStr

    # Tunables with bounds
    max_workers: int = Field(default=4, ge=1, le=64)
    request_timeout_s: float = Field(default=30.0, gt=0)

    # Mode
    debug: bool = False
    environment: Literal["dev", "staging", "prod"] = "dev"
```

What you get for free:
- **Validation at startup.** If `DATABASE_URL` isn't a valid Postgres DSN, the process refuses to start. You learn at deploy, not in the middle of a request.
- **Type coercion.** `MAX_WORKERS=8` (a string from the env) becomes `int(8)`.
- **`.env` file support.** Same `Settings` class works in local dev and production.
- **`SecretStr` in repr.** Logs show `secret_key=SecretStr('**********')`, not the actual key.

---

## Load once, at startup

Construct one `Settings` instance at the entry point. Pass it down. Never re-instantiate.

```python
# main.py
from .config import Settings
from .app import build_app

settings = Settings()                    # validated here; crashes the process if invalid
app = build_app(settings)
```

```python
# app.py
def build_app(settings: Settings) -> FastAPI:
    app = FastAPI(debug=settings.debug)
    app.state.settings = settings
    app.state.db = create_engine(str(settings.database_url))
    return app
```

```python
# routes.py — receives settings via DI, never constructs its own
@app.get("/things")
def list_things(settings: Settings = Depends(get_settings)) -> list[Thing]:
    return query_things(timeout=settings.request_timeout_s)
```

---

## Layered loading: defaults < `.env` < env vars < CLI flags

The layering happens automatically with `pydantic-settings`:

1. **Class defaults** — `debug: bool = False` in the model.
2. **`.env` file** — committed `.env.example`; uncommitted `.env` for local secrets.
3. **Process environment** — `export DEBUG=true` overrides `.env`.
4. **CLI flags** — for scripts and tools, layered on top via your CLI parser.

```python
# .env (NOT committed)
DATABASE_URL=postgresql://localhost/mydb_dev
SECRET_KEY=devkey-not-for-prod
DEBUG=true

# .env.example (committed)
DATABASE_URL=postgresql://user:password@host:5432/db
SECRET_KEY=
DEBUG=false
```

For CLI tools, parse args first and override with `Settings(**cli_overrides)`:

```python
def main() -> None:
    args = parse_args()
    settings = Settings(debug=args.debug) if args.debug is not None else Settings()
    run(settings)
```

---

## Never read `os.environ` directly in business code

This is the rule that fails most. As soon as one function does `os.environ.get("FEATURE_X")`, the discipline collapses.

```python
# BAD — config scattered, untyped, untested
def send_email(to: str, body: str) -> None:
    api_key = os.environ.get("SENDGRID_API_KEY")
    if not api_key:
        raise RuntimeError("SENDGRID_API_KEY not set")
    timeout = int(os.environ.get("EMAIL_TIMEOUT", "30"))
    ...

# GOOD — typed config, injected, tested by overriding
@dataclass
class EmailService:
    api_key: SecretStr
    timeout_s: float

    def send(self, to: str, body: str) -> None: ...

# Wiring (once, at startup)
email_service = EmailService(
    api_key=settings.sendgrid_api_key,
    timeout_s=settings.email_timeout_s,
)
```

The only place that touches `os.environ` is `BaseSettings`. Everywhere else is plain Python receiving typed values.

---

## Secrets

### `SecretStr` for any credential

```python
class Settings(BaseSettings):
    database_password: SecretStr
    stripe_api_key: SecretStr
    jwt_signing_key: SecretStr
```

`SecretStr` wraps the value so `repr()`, `str()`, and JSON serialization show `**********`. To use the underlying value: `settings.stripe_api_key.get_secret_value()`. The explicit unwrap is the point — it shows up in code review.

### Pull secrets from a real secret manager in production

`.env` is fine for development. In production, point `BaseSettings` at AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, or Kubernetes secrets via a custom settings source:

```python
class Settings(BaseSettings):
    database_password: SecretStr
    stripe_api_key: SecretStr

    @classmethod
    def settings_customise_sources(cls, *args, **kwargs):
        return (
            init_settings,
            env_settings,
            aws_secrets_source,     # custom: pulls from AWS Secrets Manager
            dotenv_settings,
            file_secret_settings,
        )
```

The model interface stays identical. Tests can still pass plain `SecretStr("test-key")`.

### Never commit secrets

- `.env` is in `.gitignore`. Always.
- `.env.example` is committed with empty values.
- Pre-commit hook with `detect-secrets` or `gitleaks`.

---

## Per-environment overrides — no `if env == "prod"`

When config differs by environment, **don't branch in code**. Branch in the config files.

```python
# BAD
if settings.environment == "prod":
    log_level = "WARNING"
    db_pool_size = 20
else:
    log_level = "DEBUG"
    db_pool_size = 2

# GOOD — different .env per environment, loaded by deploy tooling
# .env.dev
LOG_LEVEL=DEBUG
DB_POOL_SIZE=2

# .env.prod
LOG_LEVEL=WARNING
DB_POOL_SIZE=20
```

Code reads `settings.log_level` and `settings.db_pool_size`. The environment isn't visible in business logic. If `settings.environment` appears in an `if`, it's almost always a missing config knob.

---

## Validation: fail at startup, never mid-request

Use Pydantic's validators to enforce invariants the type system can't:

```python
from pydantic import field_validator, model_validator

class Settings(BaseSettings):
    database_url: PostgresDsn
    cache_ttl_s: int = Field(default=300, ge=0)
    debug: bool = False
    secret_key: SecretStr

    @field_validator("secret_key")
    @classmethod
    def secret_key_min_length(cls, v: SecretStr) -> SecretStr:
        if len(v.get_secret_value()) < 32:
            raise ValueError("secret_key must be at least 32 characters")
        return v

    @model_validator(mode="after")
    def debug_off_in_prod(self) -> Self:
        if self.environment == "prod" and self.debug:
            raise ValueError("debug must be False in prod")
        return self
```

If `secret_key` is 8 characters, the process refuses to start. You find out at deploy, not when a customer's session gets hijacked.

---

## Testing with config

Don't monkeypatch env vars. Pass a `Settings` instance directly.

```python
# BAD — fragile, leaks state across tests
def test_send_email(monkeypatch):
    monkeypatch.setenv("SENDGRID_API_KEY", "test-key")
    monkeypatch.setenv("EMAIL_TIMEOUT", "5")
    send_email("a@b.c", "hi")

# GOOD — explicit
def test_send_email():
    service = EmailService(
        api_key=SecretStr("test-key"),
        timeout_s=5.0,
    )
    service.send("a@b.c", "hi")
```

For full-app integration tests, construct a test `Settings`:

```python
@pytest.fixture
def test_settings() -> Settings:
    return Settings(
        database_url="postgresql://localhost/test_db",
        secret_key=SecretStr("x" * 32),
        debug=True,
        environment="dev",
    )

def test_app_starts(test_settings):
    app = build_app(test_settings)
    assert app.debug is True
```

---

## Anti-patterns

### Global singletons accessed from anywhere

```python
# BAD
# config.py
settings = Settings()

# somewhere_random.py
from .config import settings           # implicit dependency, untestable
def do_thing():
    timeout = settings.request_timeout_s
```

Hidden global state is hidden coupling. Pass `Settings` (or the specific values you need) explicitly. Use DI.

### Mutable config

```python
# BAD
settings.debug = True                  # runtime mutation; surprises everywhere

# GOOD — frozen at construction
class Settings(BaseSettings):
    model_config = SettingsConfigDict(frozen=True, ...)
```

Config is set at startup and never changes during the process's life. If something needs to change at runtime (feature flag, rate limit), that's a **feature flag** or a **runtime value**, not config. Put it in a different system (LaunchDarkly, a database row).

### `dict`-shaped config

```python
# BAD — strings, no types, no validation, no autocomplete
CONFIG = {
    "database_url": "...",
    "timeout": 30,
    "debug": True,
}

def f():
    db = connect(CONFIG["database_url"])      # typo? KeyError at runtime
```

```python
# GOOD — Settings instance
db = connect(str(settings.database_url))      # typo → mypy/pyright error
```

### One giant `Settings` class for everything

If `Settings` has 80 fields covering five subsystems, split it.

```python
class DatabaseSettings(BaseSettings):
    url: PostgresDsn
    pool_size: int = 10
    model_config = SettingsConfigDict(env_prefix="DB_")

class EmailSettings(BaseSettings):
    api_key: SecretStr
    timeout_s: float = 30.0
    model_config = SettingsConfigDict(env_prefix="EMAIL_")

class Settings(BaseSettings):
    database: DatabaseSettings = DatabaseSettings()
    email: EmailSettings = EmailSettings()
    debug: bool = False
```

`DB_URL=...` and `EMAIL_API_KEY=...` route to the right sub-settings. Each subsystem can receive only what it needs.

---

## Quick reference

| Symptom | Fix |
|---|---|
| `os.environ.get(...)` in business code | Move to `Settings`, inject |
| Secret in `repr` / logs | `SecretStr` |
| Config typo only caught at runtime | Typed `Settings` field |
| `if env == "prod"` in code | Different config value per environment |
| Test that mutates env vars | Pass a test `Settings` instance |
| Settings class with 80 fields | Split into sub-settings with `env_prefix` |
| Config silently accepts unknown keys | `extra="forbid"` |
| Runtime mutation of `settings.x` | Frozen settings + feature flag system |
