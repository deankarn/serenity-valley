---
name: clean-code
description: Everyday code conventions — container-relative names (users.Get, not userStore.GetUser), get returns an optional and find a collection, arguments ordered by variance and named to match across calls, booleans as questions, distinct types where mix-ups are costly, enums instead of flag arguments, early returns, comments that say why, and never fighting the project's formatter. Use when writing, reviewing, or refactoring code, and when naming or designing functions, methods, parameters, or variables. Invoke directly as /clean-code verify to audit a plan or codebase against these conventions.
when_to_use: Trigger when writing or editing code in any language, naming or designing a function, method, parameter, variable, or type, writing a data lookup or repository method, adding an ID, boolean, or duration parameter, writing comments, or reviewing a diff. Trigger on phrasings like "what should I name", "clean up this code", "is this naming right", "refactor this function". Do not trigger for prose, configuration, or dependency bumps.
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

## Naming

**Let the container say it.** The container the caller sees — Go package, Rust module or type, Java class — already names the noun. Do not repeat it: `user.Store.Get`, not `user.Store.GetUser`; Java `UserStore.get`, not `UserStore.getUser`. Name the variable holding an instance after its container, plural when it holds many: `users`, not `userStore` or `userDAO`. The call site reads `users.Get(id)`, and `user` stays free for the value.

**Lookups: `get` or `find`.**

- **`get`** returns zero or one — an optional.
- **`find`** returns zero or more — a collection, empty when nothing matches.
- Absence is never an error (see `eng:box-design` error contracts). Errors are for real failures.
- The doc comment states the return shape: "or None if there isn't one", "empty if there are none".
- The rest of the name follows the container rule: `Get`, `FindByTeam`.

| Language | `get` | `find` |
|---|---|---|
| Go | `(Option[T], error)` | `([]T, error)` |
| Rust | `Result<Option<T>, E>` | `Result<Vec<T>, E>` |
| Java | `Optional<T>` | `List<T>` (or another collection) |

Go's `Option` is `github.com/go-playground/pkg/v5/values/option`, dot-imported — `. "github.com/go-playground/pkg/v5/values/option"` — so `Option`, `Some`, and `None` read like the standard library. It beats a nil pointer: a nil check is not enforced, and nil can be a valid value. Where a project already has a convention — pointer and nil, or another Option type — follow it, and do not add the dependency unprompted. New projects default to it.

Applies to lookups from a store, service, or collection. Plain field accessors are exempt.

**Name arguments, then match them.** Parameters get specific names. A caller's variable takes the name of the parameter it is passed to — `users.Get(ctx, id)` with `id`, not `x` or `uid` — so a value keeps one name through the call chain. Exception: truly generic code (`slices.Contains(s, v)`), whose parameter names are generic by design.

**Booleans are questions:** `isActive`, `hasAccess`, `shouldRetry` — never `active` or `access`.

**Name for the viewport.**

- Short names, even single letters, are fine when the variable's whole scope fits within about 40 lines — declaration and every use visible at once.
- Loop counters may always be single letters.
- Beyond that, name for what is happening: specific but short. `retries`, not `r`, and not `numberOfRetryAttemptsSoFar`.
- Functions grow; when in doubt, start specific.

**If it can crash, say so.** A function that panics or throws to stop the application names or documents it: Go `Must*`, Rust `expect` with a `# Panics` doc section, Java `require*`. Use them at startup and for defects only (see `eng:box-design` error contracts) — never on a request path.

## Types

**Give concepts their own types — only where it pays.** New types cost code, framework glue, and coordination (where the type lives, who owns it, who must agree), which slows development. Use them only for a tangible benefit:

- sensitive operations where a swap does real damage — moving money, cancelling or deleting, granting permissions, crossing customers or tenants — especially where same-primitive IDs meet in one signature (`userID, orderID int64`);
- values with rules worth enforcing once (money, an `Email` validated in its constructor);
- units (below) — standard types, so no coordination cost.

Elsewhere, primitives are fine. The bar is the same in every language: the code is cheaper in Go and Rust, but the coordination costs the same. Never retrofit a codebase just for this.

This is an optional recommendation, not a rule. When those conditions hold, suggest the type to the user and let them decide; do not introduce new value types unprompted. The rules below apply once a type exists.

- Go: a defined type, `type UserID int64`. Never an alias (`type UserID = int64`), which protects nothing. `database/sql` accepts and scans defined types directly.
- Rust: a newtype, `struct UserId(i64)`, with `From<i64>` in and `Deref<Target = i64>` out. serde's and sqlx's `transparent` attributes take the newtype directly.
- Java: a record, `record UserId(long value)`; `.value()` is the way out, and each framework (Jackson, JPA, jOOQ) is taught the type once.
- Other languages: their equivalent — Kotlin value classes, TypeScript branded types, Python `NewType`.

Convert at the edges only: raw to typed where values enter (request parsing, row reading), typed to raw where they leave (storage writes). Business logic never converts.

Placement — these types are shared vocabulary, not payloads. Who defines the meaning decides where they live:

- One box defines it (`UserID`): with that box's contract, e.g. `user.ID` next to `user.Store`.
- Nobody owns it and everyone agrees on it (email, money): the shared foundation library, alongside `Retryable` (see `eng:retry-contracts`).
- A standard or well-known library type exists (`time.Duration`, `java.time`): use it instead of writing one.

The type stays dependency-free — no JSON or ORM annotations. Framework glue (JPA converter, Jackson mixin, custom scanner) lives in the box that uses the framework.

Do not wrap everything. Values that cannot be confused with another and carry no rules stay primitive.

**Units belong in types.** `timeout time.Duration` / `Duration`, never `timeoutMs int`. No unit suffixes on plain numbers.

## Functions

**Order arguments by variance**, least to most:

1. Always-present values — Go's `ctx context.Context`.
2. Values that barely vary — constants, configuration, enums.
3. Values that vary most — the payload.

`notifier.Send(ctx, Urgent, msg)`: call sites line up, and attention lands on the argument that differs. Positions the language fixes win — receivers and `self` first, variadic arguments and trailing closures last.

**No flag arguments.** Replace a boolean parameter with an enum: `Send(ctx, Urgent, msg)`, not `Send(ctx, msg, true)`. The enum is low-variance, so it goes before the payload.

**Return early.** Guard clauses first, each returning immediately; the happy path stays unindented. No nested `if` pyramids.

## Comments

- Comments explain *why* — intent, a constraint, a workaround. Never restate *what* the code does.
- Doc comments state the contract: return shape, what absence looks like, whether it can panic.
- No commented-out code. Delete it; git has history.

## Formatting and linting

Which formatter and linter a project uses, and how, is the project's decision. Respect it:

- Run the project's formatter on the files you touch. Never hand-format.
- Never reformat code you did not otherwise change as part of a logic change.
- Never add or change formatter or linter configuration, or suppress a lint, unless asked.
- Never raise formatting in review.

## When planning

Interfaces in the plan use container-relative names, `get`/`find` for lookups, distinct ID and unit types, variance-ordered parameters, and enums instead of boolean flags.
