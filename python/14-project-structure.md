# Project Structure & Tooling

How to lay out a Python project so adding a file is obvious, running tests is one command, and CI catches what humans miss.

---

## The layout

Two layouts are common; pick based on whether the project is **packaged** (built into a wheel, installed, distributed) or **deployed as-is** (containerized service, internal app, script).

### `src/` layout — for packaged code (libraries, CLI tools, anything you `pip install`)

```
my-project/
├── src/
│   └── my_project/                 ← the package; underscores (dashes aren't valid Python identifiers)
│       ├── __init__.py
│       ├── api/
│       ├── billing/
│       ├── config.py               ← Settings lives here
│       └── main.py                 ← entry point
├── tests/
│   ├── conftest.py
│   ├── api/
│   └── billing/                    ← mirrors src/my_project/
├── pyproject.toml
├── uv.lock
└── README.md
```

Why `src/`: importing `my_project` from the repo root **fails** until you install the package. That forces tests to run against the installed wheel — same code path as production. Packaging bugs (missing files in `pyproject.toml`'s package config, missing `__init__.py`) surface at install time, not in production.

Default for libraries, CLI tools, and any project that ships a wheel.

### Flat layout — for deployed services and scripts

```
my-project/
├── my_project/                     ← package at the repo root
│   ├── __init__.py
│   ├── api/
│   ├── billing/
│   └── main.py
├── tests/
├── pyproject.toml
├── uv.lock
└── README.md
```

Fine when the **codebase is the deploy artifact** — containerized web services, scripts, internal apps where you `uvicorn my_project.main:app` (or equivalent) and never build a wheel. Simpler — no extra directory level — and no packaging-correctness concerns because nothing is being packaged.

### How to decide

| Project | Use |
|---|---|
| Library on PyPI | `src/` |
| CLI tool distributed via `pip install` / `uv tool install` | `src/` |
| Web service in a container, deployed via Docker / k8s | flat is fine; `src/` also works |
| Internal app, never installed | flat |
| Single-script tool, notebooks, data exploration | flat (or no package at all) |
| You're not sure | `src/` — slightly more setup, catches more bugs |

The rest of this file is layout-agnostic — `pyproject.toml`, `uv`, `ruff`, `pyright`, `pre-commit` all work identically either way.

### Other conventions, both layouts

**Package name:** underscores, lowercase. Dashes aren't valid in Python identifiers — `import my-project` is a syntax error. The distribution name in `pyproject.toml` (`name = "my-project"`) can use dashes; the directory on disk must use underscores.

**`tests/` outside the package**, not `src/my_project/tests/` or `my_project/tests/`. Tests don't ship.

**`tests/` mirrors the package.** `my_project/billing/invoice.py` is tested by `tests/billing/test_invoice.py`. One test file per module.

**Other directories:**

```
my-project/
├── scripts/                        ← one-off scripts, not shipped
├── docs/                           ← optional; mkdocs / sphinx
├── .env.example                    ← committed; .env is gitignored
├── .gitignore
└── .pre-commit-config.yaml
```

---

## `pyproject.toml` — the single source of truth

Everything tooling-related goes here. No more `setup.py`, `setup.cfg`, `Pipfile`, `requirements.txt`, `mypy.ini`, `pytest.ini`, `tox.ini`. One file.

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "What this project does in one sentence."
requires-python = ">=3.12"
readme = "README.md"
license = {text = "MIT"}
authors = [{name = "You", email = "you@example.com"}]

dependencies = [
    "fastapi>=0.110",
    "pydantic>=2.6",
    "pydantic-settings>=2.2",
    "sqlalchemy>=2.0",
    "httpx>=0.27",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.1",
    "ruff>=0.4",
    "pyright>=1.1",
    "pre-commit>=3.7",
]

[project.scripts]
my-app = "my_project.main:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_project"]
```

Everything below is tool config in the same file (`ruff`, `pyright`, `pytest`, `coverage`). One place to look.

---

## `uv` — the package manager

Use `uv`. It's 10–100× faster than `pip` + `pip-tools`, handles Python versions, locks deterministically, and replaces `pyenv` / `poetry` / `pip-tools` for most workflows.

```bash
uv venv                              # create .venv/
uv sync                              # install from pyproject + uv.lock
uv sync --extra dev                  # include dev deps
uv add httpx                         # add a runtime dep
uv add --dev pytest-cov              # add a dev dep
uv remove some-package
uv run pytest                        # run a command in the venv
uv lock --upgrade                    # bump all to latest within constraints
```

**Commit `uv.lock`.** It's the deterministic source of truth — same lockfile = same install on every machine and CI run. Without it, `pip install` resolves fresh every time and you get drift.

For applications, pin tight (`fastapi>=0.110,<0.111`). For libraries, pin loose (`fastapi>=0.110`). The lockfile handles exact versions for apps; libraries shouldn't constrain consumers.

---

## `ruff` — linter + formatter

Replaces `black`, `isort`, `flake8`, `pylint`, `pyupgrade`, and a dozen plugins. One tool, one config.

```toml
[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests"]

[tool.ruff.lint]
# Start strict. Loosen only with cause.
select = [
    "E", "W",        # pycodestyle
    "F",             # pyflakes
    "I",             # isort
    "N",             # naming
    "UP",            # pyupgrade
    "B",             # bugbear
    "A",             # builtins shadowing
    "C4",            # comprehensions
    "DTZ",           # timezone-aware datetimes
    "T20",           # no print
    "RET",           # return statements
    "SIM",           # simplifications
    "TCH",           # type-checking blocks
    "ARG",           # unused args
    "PTH",           # pathlib over os.path
    "ERA",           # eradicate commented-out code
    "PL",            # pylint subset
    "RUF",           # ruff-specific
]
ignore = [
    "E501",          # line length (formatter handles it)
    "PLR0913",       # too many args (we use config dataclasses; see 02-functions.md)
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["T20", "PLR2004"]      # allow print and magic numbers in tests

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

Run: `ruff check` to lint, `ruff check --fix` to auto-fix, `ruff format` to format. CI runs `ruff check` and `ruff format --check`; both fail the build.

---

## `pyright` — type checker

`pyright` is faster than `mypy` and stricter by default. Use it.

```toml
[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "strict"
include = ["src", "tests"]
exclude = ["**/__pycache__"]

reportMissingTypeStubs = "warning"
reportUnknownParameterType = "error"
reportUnknownArgumentType = "error"
reportUnknownVariableType = "error"
reportUnknownMemberType = "warning"
reportUntypedFunctionDecorator = "error"
reportPrivateUsage = "warning"
reportUnusedImport = "error"
reportUnusedVariable = "error"
```

`strict` mode catches everything `basic` would, plus untyped function parts. New projects should start here.

For existing untyped projects: start with `typeCheckingMode = "basic"`, fix the errors, then bump to `strict`. Don't try to flip the switch on a 200K-line untyped codebase in one PR.

---

## `pytest` — test runner

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = [
    "-ra",                           # show short summary for all except passed
    "--strict-markers",              # error on unknown markers
    "--strict-config",
    "--cov=src",
    "--cov-report=term-missing",
    "--cov-fail-under=80",
]
asyncio_mode = "auto"                # pytest-asyncio: no @pytest.mark.asyncio needed
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks integration tests",
]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "raise NotImplementedError",
    "if TYPE_CHECKING:",
    "if __name__ == .__main__.:",
]
```

`--cov-fail-under=80` enforces a floor. Tune up as the project matures; never let it drift down without a reason.

---

## `pre-commit` — never push code that hasn't been checked

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict
      - id: detect-private-key

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/RobertCraigie/pyright-python
    rev: v1.1.360
    hooks:
      - id: pyright

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: uv run pytest -x --no-cov
        language: system
        types: [python]
        pass_filenames: false
```

```bash
uv run pre-commit install            # one-time, per clone
uv run pre-commit run --all-files    # run on everything (for first install)
```

Every commit now runs ruff, pyright, and pytest. CI runs the same hooks — if pre-commit passes locally, CI will too.

---

## `.gitignore` — the essentials

```gitignore
# Python
__pycache__/
*.py[cod]
*.so
.Python
.venv/
venv/
build/
dist/
*.egg-info/

# Testing
.pytest_cache/
.coverage
coverage.xml
htmlcov/
.tox/

# Type checkers
.mypy_cache/
.pyright/
.ruff_cache/

# Environment
.env
.env.*
!.env.example

# IDE
.vscode/
.idea/
*.swp
.DS_Store

# Project-specific
data/
*.log
```

**Never commit `.env`.** Always commit `.env.example` with empty / placeholder values (see `09-configuration.md`).

---

## CI minimal config (GitHub Actions)

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v2
        with:
          enable-cache: true
      - run: uv sync --extra dev
      - run: uv run ruff check
      - run: uv run ruff format --check
      - run: uv run pyright
      - run: uv run pytest
```

Five steps. No conditional skips, no "this test is flaky so we run it twice". Green or red — that's the contract.

---

## Dependencies — pin tight for apps, loose for libraries

| | Apps | Libraries |
|---|---|---|
| `pyproject.toml` constraints | `httpx>=0.27,<0.28` | `httpx>=0.27` |
| `uv.lock` committed | yes | no (only matters for the install you ship) |
| Upgrade cadence | weekly `uv lock --upgrade`, review the diff | semver pinning + Dependabot |

Apps **own** their runtime — you control deploy, so deterministic builds matter. Libraries are consumed by other projects — over-constraining causes diamond dependency hell.

---

## Versioning

Use SemVer. Bump:
- **MAJOR** on breaking changes.
- **MINOR** on new functionality.
- **PATCH** on bug fixes only.

For apps, calendar versioning (`2026.05.18`) is also fine — pick one convention.

Tag the release:
```bash
git tag -a v0.2.0 -m "Add idempotency keys"
git push --tags
```

For libraries publishing to PyPI: `uv build && uv publish`.

---

## README — the contract

Every project's README has, in order:

1. **One-line description** — what is this thing, who is it for.
2. **Install** — the one command someone runs to get started.
3. **Quick example** — 10 lines of code showing the core use case.
4. **Configuration** — required env vars, with link to `.env.example`.
5. **Develop** — `uv sync`, `pre-commit install`, `pytest`. Verbatim.
6. **Architecture** — one paragraph + a folder tree, if it's a non-trivial app.
7. **Contributing / license** — link, don't paste.

Anything more than that goes in `docs/`. Anything less, and a new contributor wastes the first hour figuring out how to run the tests.

---

## Editor — `.vscode/settings.json` (if VS Code)

Optional, but pays for itself:

```json
{
  "python.defaultInterpreterPath": ".venv/bin/python",
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "python.analysis.typeCheckingMode": "strict",
  "ruff.importStrategy": "fromEnvironment"
}
```

This is per-project, committed. Editor-specific personal preferences (themes, keybindings) don't belong here.

---

## The default stack — one screen

For a new project in 2026, the answer is:

| Concern | Tool |
|---|---|
| Package manager | `uv` |
| Lint + format | `ruff` |
| Type check | `pyright` (strict) |
| Test | `pytest` + `pytest-asyncio` + `pytest-cov` |
| Pre-commit hooks | `pre-commit` |
| Build backend | `hatchling` |
| HTTP server (web) | `fastapi` + `uvicorn` |
| Config | `pydantic-settings` |
| Validation | `pydantic` |
| ORM (sync) | `sqlalchemy` 2.x |
| ORM (async) | `sqlalchemy` 2.x async, or `asyncpg` directly |
| HTTP client | `httpx` |
| CLI | `typer` |
| Logging structure | stdlib `logging` + JSON formatter, or `structlog` |

Deviate only with a reason.

---

## Quick reference

| Symptom | Fix |
|---|---|
| Library / CLI tool without `src/` | Switch to `src/` layout so tests run against the installed package |
| `setup.py` + `requirements.txt` + `Pipfile` | One `pyproject.toml` |
| `pip install -r` in CI | `uv sync` |
| No lockfile committed | `uv.lock` in version control |
| `black` + `isort` + `flake8` | `ruff check` + `ruff format` |
| `mypy` with `# type: ignore` everywhere | `pyright` strict, fix the underlying issues |
| Tests inside the package | `tests/` at repo root, mirror `src/` |
| `.env` committed | `.gitignore` it; commit `.env.example` |
| Linter passes locally but CI fails | Wire identical checks via `pre-commit` |
| README lacks install / dev commands | Add them; verbatim shell snippets |
| `pip install package` ad-hoc | `uv add package` (updates `pyproject.toml` + lockfile) |
| Pinned everything in a library | Loosen to `>=` ranges |
| Loose pinning in an app | Pin tight + commit lockfile |
