# Contributing

Thanks for considering a contribution. This repo lives or dies by the quality of the guides, so the bar is "would I want Claude to read this?".

## Adding a new language

1. Create a lowercase folder: `typescript/`, `go/`, `rust/`, etc.
2. Mirror the existing Python file layout:
   - `README.md` — overview + non-negotiables for the language
   - `01-principles.md` — the core rules (the canonical short version of the guide)
   - `02-functions.md` — function design and single-responsibility
   - `03-data-types.md` — modeling data, the type system, value vs entity
   - `04-error-handling.md` — exceptions / Result types / validation
   - `05-design-patterns.md` — language-idiomatic implementations
   - `06-refactoring-recipes.md` — symptom → cure
   - `07-modern-<lang>.md` — modern features to prefer
3. Open a PR.

Not every language needs every file — Go probably doesn't need a "patterns" file the way Python does. Use judgment, but keep the file numbering so the reading order stays consistent.

## Voice and style of the guides

The non-negotiables for the content itself:

- **Prescriptive.** "Do X. Never Y." Not "consider X" or "you might want Y."
- **Example-driven.** Every rule has a before/after code example. No abstract prose.
- **Dense.** No padding. No "in this section we will…". Every line is rule content.
- **Modern.** Default to the latest stable language features. Don't waste lines on legacy syntax — call it out as the smell, not as the baseline.
- **Self-contained.** A reader who lands on `02-functions.md` should understand it without reading the rest. Cross-link, but don't require prerequisites.

If a section reads like a tutorial, rewrite it as rules + examples.

## Improving an existing guide

- Found a rule that's wrong, outdated, or controversial in your language community? Open an issue with reasoning.
- Spotted a missing recipe in `06-refactoring-recipes.md`? PR it — those are easy to add.
- Want to revise the prose for clarity? Welcome, but keep the prescriptive voice.

## Style mechanics (markdown)

- Use fenced code blocks with the language tag (` ```python `, ` ```ts `).
- Tables for comparison / lookup, not for body content.
- Headers in sentence case, not Title Case.
- Use `**Rule:**` to highlight prescriptive lines at the start of a rule.
- One blank line between sections. Don't double-blank.

## PR checklist

- [ ] All code examples are syntactically valid in the target language.
- [ ] Before/after pairs actually demonstrate the rule.
- [ ] Voice is prescriptive — no "you might want to consider".
- [ ] No marketing fluff or anecdotes.
- [ ] Cross-references between files use relative links.

## License

By contributing, you agree your contribution is licensed under the [MIT License](LICENSE).
