# Top-Tier Python Style Guide

A prescriptive style guide distilled from a deep read of [ArjanCodes/examples](https://github.com/ArjanCodes/examples) (years 2022–2026). Optimized for use as a Claude system prompt or `CLAUDE.md` reference. When these rules are followed, Python code is small, testable, type-safe, and reads like prose.

## How to use this

- **As a CLAUDE.md for a Python project**: copy the contents of `01-principles.md` into the project's `CLAUDE.md`. For deeper guidance, point Claude at this whole folder.
- **As a code-review checklist**: walk `01-principles.md` line by line; jump to the deep dive for any rule that needs more.
- **As a refactoring reference**: when you spot a smell, look it up in `06-refactoring-recipes.md` — every entry is "symptom → cure".

## The files (read in order on first pass)

| File | What's in it |
|---|---|
| `01-principles.md` | The 12 non-negotiable rules, plus cohesion/coupling and a SOLID rosetta. Start here. |
| `02-functions.md` | Function design and single-responsibility. **The most important file.** |
| `03-data-types.md` | Dataclasses, Pydantic, Enum, Protocol, generics, `@property`. |
| `04-error-handling.md` | Custom exceptions with context, Result types, precondition validation. |
| `05-design-patterns.md` | Repository, Factory, Builder, Strategy, DIP+DI, Template Method, Bridge, MVC, CQRS. |
| `06-refactoring-recipes.md` | 29 symptom → cure recipes. When you see X, do Y. |
| `07-modern-python.md` | PEP 695 generics, walrus, `Self`, `removeprefix`, `X \| None`, `match`. |
| `08-testing.md` | pytest, AAA, parametrize, fixtures, fakes-over-mocks, refactoring for testability. |
| `09-configuration.md` | `pydantic-settings`, layered loading, `SecretStr`, per-env overrides, anti-patterns. |
| `10-logging.md` | Logger-per-module, level semantics, structured context, JSON in prod, correlation IDs. |
| `11-dates-money.md` | `Decimal` for money, `Money(amount, currency)`, timezone-aware `datetime`, `zoneinfo`, ISO 8601 at boundaries. |

The raw study notes (descriptive, not prescriptive) live at `../ArjanCodes-Patterns.md`.

## The 12 non-negotiables (one-line each)

1. **One function, one thing.** Can't describe it without "and"? Split it.
2. **Type-hint every signature.** No exceptions for public functions.
3. **No `isinstance` chains for dispatch.** Use polymorphism, `match`, or a dict.
4. **No boolean flag parameters.** Two functions, not one with a `flag: bool`.
5. **No magic strings.** Use `Enum`.
6. **No wildcard imports.** Ever.
7. **Validate at boundaries, trust internally.** Pydantic at the edge, dataclasses inside.
8. **Frozen dataclasses for value objects.** Mutation is opt-in, not default.
9. **Custom exceptions carry context.** Attributes on the exception, not parsed from `str(e)`.
10. **Names carry documentation.** Verb-object, full words, no type info in the name.
11. **List comprehensions over `for ... append`.** With filter clauses.
12. **Dict lookup over list scan.** O(1) over O(n), always.

## The unifying philosophy

> **Make illegal states unrepresentable, push variation into types, and write functions small enough that you can describe each one without using the word "and".**

If a rule in this guide conflicts with that sentence, the sentence wins.
