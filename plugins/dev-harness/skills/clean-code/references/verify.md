# Verify

Pick the checklist by what is in context. A proposed plan or interface design → **Plan audit**. Source files, a diff, or a repo → **Code audit**. Both present → run both, plan first.

State which one you ran. Report findings grouped by file or interface, each with a severity and the specific location. Explicitly list what is already correct and should be left alone.

Severity:
- **convention** — breaks one of the conventions. Fix before merging.
- **note** — smell, judgement call, or something to watch.

Give each finding a one-line fix direction. Do not rewrite code during an audit unless asked. Report, then offer.

---

## Plan audit

1. Are method names container-relative — no repeating the package, module, or class noun?
2. Are data lookups `get` (zero or one, optional) and `find` (zero or more, collection)?
3. Is absence modelled as an empty optional or collection, never as an error?
4. Are parameters ordered by variance — always-present first, payload last — with enums instead of boolean flags?
5. Where same-primitive concepts meet or values carry rules, do they have their own types — converted only at the edges, placed with their owner or in the shared foundation library?

---

## Code audit

Use Grep and Glob. Adapt the patterns to the language — these are signals, not literal searches.

**Naming**
- Stutter: a method or type repeating its container's noun (`user.Store.GetUser`, `UserStore.getUser`, `user::UserStore`).
- Instance variables not named after their container (`userStore`, `userDAO`, `repo` for a store of users that should be `users`).
- `get` returning a collection, or throwing or erroring when nothing is found.
- `find` returning a single item, a nullable, or nil.
- Lookups returning a bare pointer or null for absence where the project has an Option type.
- Lookups whose doc comment does not state the return shape.
- Call-site variables renamed from the parameter they feed (`uid` passed as `id`) without reason. Generic code is exempt.
- Booleans not phrased as questions (`active`, `admin`, `retry` instead of `isActive`, `isAdmin`, `shouldRetry`).
- Terse names (one or two letters, cryptic abbreviations) whose scope spans more than about 40 lines. Loop counters are exempt.
- Names longer than the concept needs.
- Functions that panic or throw to stop the application without saying so — no `Must*`, `# Panics`, or `require*` — or such functions on a request path.

**Types**
- Two or more parameters of the same primitive type representing different concepts (`userID int64, orderID int64`) — a swap compiles.
- A Go type alias (`type UserID = int64`) where a defined type (`type UserID int64`) was meant.
- Conversions between raw and typed values inside business logic rather than at the edges.
- Units in names on plain numbers (`timeoutMs int`, `delaySeconds`) instead of a duration or unit type.
- Wrapper types for values that cannot be confused and carry no rules — over-typing.
- Framework annotations (JSON, ORM) or framework dependencies on shared value types instead of glue in the edge box.
- Owner-less value types (email, money) defined inside one box instead of the shared foundation library, or hand-written where a standard type exists.

**Functions**
- Parameters out of variance order: payload before enums or configuration, or Go's `ctx` not first. Language-fixed positions are exempt.
- Boolean flag parameters.
- Nested happy paths where guard clauses would flatten them.

**Comments**
- Comments that restate what the code does.
- Commented-out code.

**Formatting**
- Reformatting of untouched code mixed into a logic change. Which formatter and settings the project uses is its decision; do not audit them.
