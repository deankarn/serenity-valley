---
name: clean-code
description: Everyday code conventions — get* returns an optional and find* returns a collection, variable names as short as their visible scope allows, and never fighting the project's formatter. Use when writing, reviewing, or refactoring code, and when naming functions, methods, or variables. Invoke directly as /clean-code verify to audit a plan or codebase against these conventions.
when_to_use: Trigger when writing or editing code in any language, naming a function, method, or variable, writing a data lookup or repository method, or reviewing a diff. Trigger on phrasings like "what should I name", "clean up this code", "is this naming right". Do not trigger for prose, configuration, or dependency bumps.
argument-hint: "[verify]"
arguments: [mode]
allowed-tools: Read Grep Glob
---

# Clean code

Mode: `$mode`

- Empty mode — apply the conventions below to the code being written or changed. Do not run an audit.
- `verify` — read `${CLAUDE_SKILL_DIR}/references/verify.md` and run the audit it defines against whatever is in context. Audit only when the user asked for it.

## Precedence

These conventions are general. Project and organization guidance wins on specifics. Apply these conventions where the project is silent, and flag genuine conflicts to the user rather than resolving them silently.

## Data lookup naming

- **`get*` returns zero or one** — an optional.
- **`find*` returns zero or more** — a collection, empty when nothing matches.
- Absence is never an error (see `dev-harness:box-design` error contracts). Errors are for real failures.
- The doc comment states the return shape: "or None if there isn't one", "empty if there are none".

| Language | `get*` | `find*` |
|---|---|---|
| Go | `(Option[T], error)` | `([]T, error)` |
| Rust | `Result<Option<T>, E>` | `Result<Vec<T>, E>` |
| Java | `Optional<T>` | `List<T>` (or another collection) |

Go's `Option` is `github.com/go-playground/pkg/v5/values/option`, dot-imported — `. "github.com/go-playground/pkg/v5/values/option"` — so `Option`, `Some`, and `None` read like the standard library. It beats a nil pointer: a nil check is not enforced, and nil can be a valid value. Where a project already has a convention — pointer and nil, or another Option type — follow it, and do not add the dependency unprompted. New projects default to it.

Applies to lookups from a store, service, or collection. Plain field accessors are exempt.

## Variable names

- Short names, even single letters, are fine when the variable's whole scope fits within about 40 lines — declaration and every use visible at once.
- Loop counters may always be single letters.
- Beyond that, name for what is happening: specific but short. `retries`, not `r`, and not `numberOfRetryAttemptsSoFar`.
- Functions grow; when in doubt, start specific.

## Formatting and linting

Which formatter and linter a project uses, and how, is the project's decision. Respect it:

- Run the project's formatter on the files you touch. Never hand-format.
- Never reformat code you did not otherwise change as part of a logic change.
- Never add or change formatter or linter configuration, or suppress a lint, unless asked.
- Never raise formatting in review.

## When planning

Name lookups `get*` and `find*` in the plan's interfaces.
