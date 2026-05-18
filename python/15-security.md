# Security

The Python-specific mistakes that turn into CVEs. Most aren't subtle — they're forgetting to do the boring correct thing.

---

## The principle: trust no input

Anything from outside the process is hostile until validated. HTTP request bodies, query strings, headers, file uploads, env vars, database rows from another service, message-bus payloads, even filenames on disk.

The discipline:

1. **Validate at the boundary** with a Pydantic model. Reject malformed input loudly.
2. **Inside the app, types are trusted.** No `if user_input.startswith("../"):` halfway down a call chain — that means the boundary failed.
3. **Output that crosses a boundary is encoded for that boundary.** SQL → parameterized. Shell → list args. HTML → escaped. URL → quoted.

---

## SQL injection — parameterize. Always.

Never build SQL with string operations. Pass values as parameters.

```python
# BAD — string interpolation; classic SQLi
cur.execute(f"SELECT * FROM users WHERE email = '{email}'")
cur.execute("SELECT * FROM users WHERE email = '" + email + "'")
cur.execute("SELECT * FROM users WHERE email = %s" % email)

# GOOD — parameterized
cur.execute("SELECT * FROM users WHERE email = %s", (email,))

# SQLAlchemy ORM — safe by construction
session.scalars(select(User).where(User.email == email)).first()

# SQLAlchemy Core text() — parameterize with :name
session.execute(text("SELECT * FROM users WHERE email = :email"), {"email": email})
```

The `%s` is **not string formatting** — the driver substitutes it with proper escaping. `f"...{x}..."` is direct string concatenation; the driver never sees it as a parameter.

**The rule:** if you have any string operation on SQL with user-controlled values in it, you have an injection. No exceptions for "I'm sure it's safe" cases.

### What about dynamic table or column names?

Parameters can only substitute *values*, not identifiers. For dynamic identifiers, **whitelist** against a known set:

```python
ALLOWED_SORT = {"created_at", "updated_at", "name"}

def list_users(sort_by: str) -> list[User]:
    if sort_by not in ALLOWED_SORT:
        raise ValueError(f"invalid sort field: {sort_by}")
    return session.execute(text(f"SELECT * FROM users ORDER BY {sort_by}")).all()
```

Never construct identifiers from raw user input even with parameters — the parameter system can't help with `ORDER BY`.

---

## Command injection — `subprocess.run` with a list, never `shell=True`

```python
# BAD — shell=True with user input; trivial command injection
import subprocess
subprocess.run(f"convert {filename} out.png", shell=True)
# user sends filename = "x.jpg; rm -rf /"

# GOOD — list of args, no shell
subprocess.run(["convert", filename, "out.png"], check=True)
```

With a list, `filename` is one argument no matter what's in it — no shell parsing happens. `shell=True` parses the string and executes shell metacharacters.

**The rule:** `shell=True` is wrong unless the entire command is a fixed string with no user input. And even then, `shell=False` is usually fine.

```python
# OK — fixed command, no user input
subprocess.run("ls -la /etc", shell=True)

# But also fine without shell:
subprocess.run(["ls", "-la", "/etc"])
```

Always `check=True` so failures raise instead of being silently ignored.

---

## Path traversal — `Path.resolve()` and verify under base

A filename from a user is not a path you can trust. `../../../etc/passwd` is a valid string.

```python
# BAD
def serve_file(filename: str) -> bytes:
    return (Path("uploads") / filename).read_bytes()

# attacker sends filename = "../../etc/passwd"

# GOOD — resolve and check containment
UPLOAD_DIR = Path("uploads").resolve()

def serve_file(filename: str) -> bytes:
    target = (UPLOAD_DIR / filename).resolve()
    if not target.is_relative_to(UPLOAD_DIR):
        raise SecurityError(f"path traversal: {filename!r}")
    if not target.is_file():
        raise FileNotFoundError(filename)
    return target.read_bytes()
```

`Path.resolve()` normalizes `..` segments. `is_relative_to(UPLOAD_DIR)` (3.9+) confirms the result stays inside the allowed base. Both checks are required — only one and you can be bypassed.

---

## Secrets — never in code, never in logs

### Don't commit secrets

- `.env` is in `.gitignore`. Always. Only `.env.example` (empty) is committed.
- Pre-commit hook with `detect-secrets` or `gitleaks`.
- If a secret ever lands in git, **rotate it**. Removing it from history is necessary but not sufficient — the value leaked the moment the commit existed on a forked clone.

### Use `SecretStr` for credentials

```python
from pydantic import SecretStr

class Settings(BaseSettings):
    database_password: SecretStr
    stripe_api_key: SecretStr
    jwt_signing_key: SecretStr
```

`SecretStr` returns `**********` from `repr()` and JSON serialization. To use the value: `settings.stripe_api_key.get_secret_value()`. The explicit unwrap is the point — it shows up in code review.

### Logs are a leak vector

```python
# BAD — Authorization header has the bearer token
log.info("request received", extra={"headers": dict(request.headers)})

# GOOD — redact sensitive headers
SENSITIVE_HEADERS = {"authorization", "cookie", "x-api-key"}
log.info("request received", extra={"headers": {
    k: v for k, v in request.headers.items() if k.lower() not in SENSITIVE_HEADERS
}})
```

See `10-logging.md` for the redaction filter pattern.

### Secrets in production

`.env` is for dev. In production, use AWS Secrets Manager / GCP Secret Manager / Vault / Kubernetes secrets, loaded via a custom `BaseSettings` source. **Never** bake secrets into Docker images, environment files committed to private repos, or CI YAML.

---

## Password hashing — `argon2id` (or `bcrypt`), never raw SHA / MD5

```python
# BAD — fast hashes; cracked in seconds with a GPU
import hashlib
digest = hashlib.sha256(password.encode()).hexdigest()
digest = hashlib.md5(password.encode()).hexdigest()         # broken since 1996

# GOOD — argon2id, the current OWASP recommendation
from argon2 import PasswordHasher

ph = PasswordHasher()                  # sensible defaults
hash_str = ph.hash(password)           # store this
ph.verify(hash_str, password)          # raises VerifyMismatchError on bad password

# Acceptable — bcrypt with work factor 12+
import bcrypt
hash_str = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
bcrypt.checkpw(password.encode(), hash_str)
```

**Never** roll your own. **Never** use `sha256` / `sha512` / `md5` for passwords — they're designed to be fast; password hashing should be deliberately slow.

Store the full hash string (`$argon2id$v=19$m=65536,t=3,p=4$...`). It contains the algorithm, parameters, and salt — everything needed to verify and to migrate when parameters change.

---

## `secrets`, not `random`, for tokens

`random` is a fast, predictable PRNG. Suitable for game logic, not for security.

```python
# BAD — predictable, seedable, recoverable from a few samples
import random
token = "".join(random.choices(string.ascii_letters + string.digits, k=32))

# GOOD — cryptographically secure
import secrets
token = secrets.token_urlsafe(32)        # 32 bytes → ~43 url-safe chars
api_key = secrets.token_hex(32)          # 64 hex chars
csrf = secrets.token_bytes(32)           # raw 32 bytes
```

Use `secrets` for:
- Session tokens, CSRF tokens, password reset links
- API keys
- Anything that an attacker should not be able to predict

---

## JWT — the footguns

JWTs are easy to misuse.

```python
# BAD — algorithm "none" accepted; attacker can forge tokens with no signature
jwt.decode(token, key="", algorithms=["none", "HS256", "RS256"])

# BAD — accepting RS256 (asymmetric) when you signed with HS256 (symmetric)
# An attacker can sign tokens with your public key as if it were the HMAC secret
jwt.decode(token, key=public_key, algorithms=["HS256", "RS256"])

# GOOD — exactly one algorithm, matching what you sign with
jwt.decode(token, key=secret, algorithms=["HS256"])
jwt.decode(token, key=public_key, algorithms=["RS256"])
```

Other JWT rules:
- **Short expiry** (`exp` claim). 15 min for access tokens; refresh tokens for longer-lived sessions.
- **Verify `aud` and `iss`** — don't accept tokens issued for a different audience.
- **Never store secrets in the payload** — JWTs are signed, not encrypted. Anyone can base64-decode and read them.
- **Rotate signing keys.** Support `kid` header for key rotation.

If you don't have a strong reason to use JWT, **don't** — server-side session IDs (a random `secrets.token_urlsafe(32)` stored in Redis) are simpler, revocable, and don't have any of these footguns.

---

## CSRF — for cookie-authenticated browser apps

If your API uses cookies for auth, you need CSRF protection. (Bearer tokens in `Authorization` headers from a JS app are CSRF-safe — browsers don't auto-send them cross-site.)

The two layers:

1. **`SameSite=Lax` (or `Strict`) cookies.** Modern browsers refuse to send these on cross-site requests. Sets a strong baseline.
2. **CSRF token** for state-changing requests. Server issues a random token, client sends it in a header (e.g., `X-CSRF-Token`). Server validates.

```python
# Setting the cookie
response.set_cookie(
    "session_id",
    value=session_id,
    httponly=True,          # JS can't read it (mitigates XSS impact)
    secure=True,            # HTTPS only
    samesite="lax",         # CSRF baseline
    max_age=86400,
)
```

`HttpOnly` is non-negotiable for session cookies. `Secure` is non-negotiable in production. `SameSite=Lax` is the minimum.

---

## XSS — escape output, never trust HTML

If you're injecting user-controlled content into HTML, **escape**. Templating engines (Jinja2, Django) auto-escape by default. **Don't disable it.**

```html
<!-- Jinja2 — autoescape on, safe -->
<p>{{ user.bio }}</p>

<!-- BAD — autoescape disabled with |safe; attacker injects <script> -->
<p>{{ user.bio | safe }}</p>
```

If you need to allow some HTML (rich-text fields), use `bleach` or `nh3` to sanitize against a strict allowlist of tags and attributes. **Never** parse HTML with regex or try to sanitize by hand.

In APIs that return JSON consumed by JS apps, XSS is a property of the JS app's rendering, not the API. Still: **never** return raw user-supplied HTML as `text/html`; always `application/json`.

---

## Open redirects — validate redirect URLs

```python
# BAD — attacker sends a link to your site with ?next=https://evil.com
@app.get("/login")
def login(next: str):
    redirect(next)

# GOOD — only allow relative paths, or whitelist hosts
def is_safe_redirect(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme or parsed.netloc:
        return False                # external URL
    return url.startswith("/")

@app.get("/login")
def login(next: str = "/"):
    target = next if is_safe_redirect(next) else "/"
    redirect(target)
```

Phishers love open redirects: the URL looks like yours, the user trusts it, but they end up at `evil.com`. Always validate.

---

## `pickle` is RCE on untrusted data

```python
# BAD — unpickling executes arbitrary Python
import pickle
data = pickle.loads(request.body)             # attacker controls request.body → RCE
```

`pickle` is fine for trusted data you produced (cache files, model weights from your own pipeline). **Never** unpickle anything that came from a network, a public API, an upload, or a queue that other systems write to.

Replacements:
- **JSON** — for structured data.
- **`msgpack`** — for binary, JSON-compatible types.
- **Protocol Buffers / Avro / FlatBuffers** — for cross-language schemas.

---

## `yaml.load` → `yaml.safe_load`

```python
# BAD — yaml.load with default Loader executes constructors → RCE
import yaml
config = yaml.load(open("config.yaml"))

# GOOD
config = yaml.safe_load(open("config.yaml"))
```

`safe_load` parses only basic types. Use it. PyYAML actually warns on `load` without an explicit loader; don't suppress the warning — switch.

---

## `eval` / `exec` on input — never

```python
# BAD — code injection
result = eval(user_input)                     # attacker → arbitrary Python
exec(request_body)                            # same

# Worse — eval on what looks like a "number"
n = eval(request.args.get("count"))           # "__import__('os').system('rm -rf /')"

# GOOD
n = int(request.args.get("count"))            # parse, don't evaluate
```

If you need to evaluate an expression DSL, write a parser (or use `ast.literal_eval` for literals only — which is safe for numbers, strings, tuples, lists, dicts, booleans, None).

```python
import ast
n = ast.literal_eval(s)                        # safe: literals only
```

---

## Dependency security

### `pip-audit` or `safety` in CI

```bash
uv run pip-audit                              # or: safety check
```

Fails the build if any installed package has a known CVE. Run on every PR.

### Dependabot (or Renovate)

Configure automatic dependency-update PRs. Most security fixes ship as patch versions; a fresh `uv sync` after a Dependabot bump is usually all that's needed.

`.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Pin transitive deps via lockfile

`uv.lock` pins the entire dependency graph. A direct dep updating its own deps doesn't silently change your install — the lockfile is the source of truth.

---

## HTTPS, HSTS, secure transport

- **No bare HTTP in production.** TLS terminates at your load balancer / ingress / CDN, but the app must redirect HTTP→HTTPS.
- **`Strict-Transport-Security` header** with `max-age=31536000; includeSubDomains; preload`.
- **Modern TLS only** — TLS 1.2+. No SSLv3, no TLS 1.0/1.1.

Most of this is handled by your load balancer config, not Python. But your app sets the HSTS header.

---

## CORS — least permissive

```python
# BAD — allow any origin with credentials → CSRF + creds-leak vector
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
)

# GOOD — explicit allowlist; credentials only when needed
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "Idempotency-Key"],
)
```

`allow_origins=["*"]` plus `allow_credentials=True` is **rejected by browsers** for security reasons — and rightly so. If you find yourself trying, you've misconfigured something.

---

## Rate limiting — defense in depth

App-level rate limiting (per-IP, per-user, per-endpoint) is your last line of defense against credential stuffing, scraping, and abuse. Even with a WAF and CDN in front, the app should enforce its own limits.

Tools: `slowapi`, `limits`, or your framework's built-in middleware. Return `429 Too Many Requests` with a `Retry-After` header (see `12-api-design.md`).

---

## SSRF — block egress to internal addresses

Server-Side Request Forgery: your server fetches a URL from user input → attacker makes you hit internal services (metadata endpoints, internal admin APIs).

```python
# BAD — fetches whatever URL the user sends
@app.post("/import")
async def import_data(url: str):
    response = await httpx.get(url)
    ...

# attacker sends url = "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
# (AWS instance metadata; leaks credentials)
```

Protections:
- **Validate the URL.** Reject IPs in private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16`, `::1`, fc00::/7).
- **Resolve the hostname first**, check the resolved IP, then fetch.
- **Disable redirects**, or follow them only after re-checking.
- **At the infra layer**, run outbound through a proxy with an egress allowlist.

```python
import ipaddress
import socket

def is_safe_url(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme not in {"http", "https"}:
        return False
    if not parsed.hostname:
        return False
    try:
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
    except (socket.gaierror, ValueError):
        return False
    return not (ip.is_private or ip.is_loopback or ip.is_link_local or ip.is_multicast)
```

This isn't bulletproof against DNS rebinding — for high-stakes apps, route all outbound traffic through a proxy with explicit allowlists.

---

## File upload security

- **Validate type via content sniffing, not extension.** Use `python-magic` to check the actual file content. `.jpg` ending doesn't mean it's an image.
- **Cap size at the framework level** (reject before reading into memory).
- **Store uploads outside the web root.** Serve them via your app, not by URL → filesystem mapping.
- **Generate the storage filename** — never use the user-provided name. `uuid4().hex + extension` works.
- **Scan with ClamAV** or a hosted equivalent if uploads are exposed to other users.

```python
import magic
from uuid import uuid4

MAX_SIZE = 10 * 1024 * 1024
ALLOWED_MIMES = {"image/jpeg", "image/png", "image/webp"}

async def save_upload(upload: UploadFile) -> Path:
    data = await upload.read(MAX_SIZE + 1)
    if len(data) > MAX_SIZE:
        raise FileTooLargeError(MAX_SIZE)

    mime = magic.from_buffer(data, mime=True)
    if mime not in ALLOWED_MIMES:
        raise UnsupportedFileTypeError(mime)

    extension = {"image/jpeg": ".jpg", "image/png": ".png", "image/webp": ".webp"}[mime]
    target = UPLOAD_DIR / f"{uuid4().hex}{extension}"
    target.write_bytes(data)
    return target
```

---

## Constant-time comparison for secrets

```python
# BAD — early-exit comparison; vulnerable to timing attacks
if api_key == expected:
    ...

# GOOD
import secrets
if secrets.compare_digest(api_key, expected):
    ...
```

`==` returns false as soon as a byte differs. Timing differences leak the prefix. `secrets.compare_digest` is constant-time. Use it for any secret-vs-secret comparison: API keys, HMACs, tokens.

---

## Anti-patterns

| Symptom | Fix |
|---|---|
| `cur.execute(f"... {value} ...")` | Parameterize: `cur.execute("... %s ...", (value,))` |
| `subprocess.run(cmd, shell=True)` with user input | List args, `shell=False`, `check=True` |
| `(base / user_filename).read_bytes()` | `resolve()` + `is_relative_to(base)` check |
| Secret in source / committed file | Move to env / secrets manager; rotate the leaked value |
| `hashlib.sha256(password)` for passwords | `argon2` or `bcrypt` |
| `random.choices(...)` for tokens | `secrets.token_urlsafe(...)` |
| `jwt.decode(..., algorithms=["none", ...])` | Single algorithm, matching what you signed with |
| Disabling Jinja autoescape | Don't; sanitize allowed HTML with `bleach` / `nh3` |
| `redirect(request.args["next"])` | Validate `next` is relative or in an allowlist |
| `pickle.loads(network_data)` | JSON / msgpack / protobuf |
| `yaml.load(...)` | `yaml.safe_load(...)` |
| `eval(user_input)` | `int(...)` / `ast.literal_eval(...)` / a real parser |
| `CORS allow_origins=["*"]` with credentials | Explicit allowlist |
| `if token == expected:` | `secrets.compare_digest(token, expected)` |
| Trusting upload extension | Sniff content with `python-magic`; generate filename |
| Fetching arbitrary user URL server-side | SSRF protections + outbound proxy |
| No `pip-audit` in CI | Add it; fail the build on CVEs |
| `HSTS` missing in production | Set `Strict-Transport-Security` |
| `Session` cookie without `HttpOnly` + `Secure` + `SameSite` | All three, always |

---

## The one-screen summary

- **Boundary validation** with Pydantic; trusted types inside.
- **Parameterize SQL.** No exceptions.
- **`subprocess` with list args**, never `shell=True`.
- **Resolve paths**, verify under base, before any file access.
- **`SecretStr` for credentials**; never log them; rotate if leaked.
- **`argon2`/`bcrypt` for passwords**, `secrets` for tokens.
- **`yaml.safe_load`, not `yaml.load`. Never unpickle untrusted data. Never `eval` user input.**
- **JWT: one algorithm, short expiry, verify aud/iss** — or use server-side sessions.
- **HTTPS-only**, HSTS header, `HttpOnly`+`Secure`+`SameSite` cookies.
- **Least-permissive CORS.** Explicit origin allowlist.
- **`pip-audit` in CI**, Dependabot for updates, commit `uv.lock`.
- **Constant-time comparison** for secrets.
