# clean-claude-code

> Prescriptive coding-style guides per language, designed to be loaded as `CLAUDE.md` or system prompts so Claude writes code that's small, readable, type-safe, and humans-actually-love-it good.

Every guide is **prescriptive** ("do X / never Y"), **example-driven** (before/after code, not abstract advice), and **dense** — every line earns its place.

---

## Why this exists

Out of the box, LLMs default to writing the most common Python on the internet — which is mediocre. You can get dramatically better output by handing the model a tight style guide. This repo is a curated set of those guides, one per language.

Plug a guide into your project's `CLAUDE.md`, or paste the principles file into a system prompt, and the code Claude writes will be noticeably cleaner.

---

## Structure

```
clean-claude-code/
├── README.md                    ← you are here
├── LICENSE
└── <language>/
    ├── README.md                ← entry point + non-negotiables
    ├── 01-principles.md         ← core rules
    ├── 02-functions.md          ← function design
    ├── ...
    └── 0N-modern-features.md    ← language-specific modern features
```

One folder per language. Each folder is self-contained — you can drop just `python/` (or just `python/01-principles.md`) into a project and get value.

---

## Languages

| Language | Status | Source / inspiration |
|---|---|---|
| **Python** | ✅ Complete | Distilled from [ArjanCodes/examples](https://github.com/ArjanCodes/examples) (2022–2026) |
| TypeScript | 🚧 Planned | — |
| Go | 🚧 Planned | — |
| Rust | 🚧 Planned | — |

PRs welcome for additional languages — see [contributing](#contributing).

---

## Quick start

### As a project `CLAUDE.md`

Pick the language. Copy the principles file into your project root as `CLAUDE.md`:

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/c0mpli/clean-claude-code/main/python/01-principles.md
```

Claude now writes Python in this style. For deeper guidance on specific topics, point it at the rest of the folder.

### As a Claude system prompt / API call

Paste the contents of `<language>/README.md` and `<language>/01-principles.md` into the system prompt of your Claude API call. ~5–10K tokens of guidance, pays for itself in every response.

### As a code-review checklist

Walk through `<language>/01-principles.md` line by line. When you spot a smell, jump to `<language>/06-refactoring-recipes.md` (or equivalent) for the fix.

---

## What's in each language guide

Every language folder contains roughly:

1. **`README.md`** — overview + non-negotiables (the short list)
2. **`01-principles.md`** — the core rules with brief examples (the long-form non-negotiables)
3. **`02-functions.md`** — function design, single-responsibility, naming
4. **`03-data-types.md`** — modeling data, the type system
5. **`04-error-handling.md`** — exceptions vs result types, validation
6. **`05-design-patterns.md`** — when and how to apply each
7. **`06-refactoring-recipes.md`** — symptom → cure table
8. **`07-modern-<language>.md`** — modern features to prefer
9. **`08-testing.md`** — how to test, with the framework that's standard for the language

Files are numbered so a single `cat */*.md` gives sensible flow.

---

## The Python guide at a glance

Twelve non-negotiables (the full version with examples is in `python/01-principles.md`):

1. **One function, one thing.** Can't describe it without "and"? Split it.
2. **Type-hint every signature.** No exceptions.
3. **No `isinstance` chains for dispatch.** Polymorphism, `match`, or a dict.
4. **No boolean flag parameters.** Two functions, not one with `flag: bool`.
5. **No magic strings.** Use `Enum`.
6. **No wildcard imports.** Ever.
7. **Validate at boundaries, trust internally.** Pydantic at the edge, dataclasses inside.
8. **Frozen dataclasses for value objects.** Mutation is opt-in.
9. **Custom exceptions carry context.** Attributes, not parsed strings.
10. **Names carry documentation.** Verb-object, full words, no type info.
11. **List comprehensions over `for ... append`.** With filter clauses.
12. **Dict lookup over list scan.** O(1) over O(n), always.

---

## Design choices in these guides

- **Prescriptive, not descriptive.** "Do X. Never Y." Not "consider X" or "you might want Y".
- **Before / after code, not theory.** Every rule has a code example showing the smell and the cure.
- **Dense.** No padding, no preamble, no "in conclusion". Every line is rule content.
- **Modern by default.** PEP 695 generics, not `TypeVar`. `X | None`, not `Optional[X]`. `pathlib`, not `os.path`.
- **Cross-referenced.** Files link to each other so you can navigate naturally.

---

## Contributing

To add a language:

1. Create a folder named after the language (lowercase): `typescript/`, `go/`, `rust/`.
2. Mirror the file structure (`README.md`, `01-principles.md`, …).
3. Keep the prescriptive, example-driven voice.
4. Open a PR.

To improve an existing guide:

- Found an outdated rule? Open an issue or PR with reasoning + code example.
- New language feature worth adopting? Add it to the relevant `07-modern-*.md`.

---

## License

[MIT](LICENSE). Use these guides freely in private or public projects, including commercial work.

---

## Credits

- **Python guide:** distilled from a deep read of [ArjanCodes/examples](https://github.com/ArjanCodes/examples) (Arjan Egges, [@arjancodes](https://github.com/arjancodes)). All credit for the underlying patterns goes to him; the distillation, prescription, and packaging are mine.
