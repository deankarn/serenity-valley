---
name: box-design
description: Decompose systems into boxes with explicit interface contracts before writing code, and audit designs or code against those contracts. Use whenever designing, planning, or architecting anything — a new service, endpoint, module, pipeline, worker, refactor, or feature — including in plan mode and when asked "how should I structure this". Invoke directly as /box-design verify to audit a plan or codebase for layering, contract, and test-boundary violations.
when_to_use: Trigger before proposing an implementation plan or writing code for anything larger than a single function. Trigger on new service, new API or endpoint, new pipeline, adding a storage layer, splitting a module, changing layering, and on phrasings like "how should I structure", "what's the best way to build", "design a", "plan the", "architecture for". Trigger on review requests mentioning coupling, layering, abstraction leakage, test boundaries, or mockability. Do not trigger for single-function edits, bug fixes, config changes, or dependency bumps.
argument-hint: "[verify|levels]"
arguments: [mode]
allowed-tools: Read Grep Glob
---

# Box design

Mode: `$mode`

- Empty mode — apply the rules below to the current design or planning work. Do not run an audit.
- `verify` — read `${CLAUDE_SKILL_DIR}/references/verify.md` and run the audit it defines against whatever is in context (a proposed plan, or the repo). Audit only when the user asked for it.
- `levels` — read `${CLAUDE_SKILL_DIR}/references/levels.md` for pitching or documenting a design at a given altitude.

## The model

Every system is a box. Zooming in turns one box into several. A box is justified by a **responsibility and a change axis**, never by file count or line count.

At implementation altitude there are three canonical kinds of box:

- **Exposure** — HTTP handler, gRPC service, CLI, queue consumer. Owns transport, validation, authn/authz wiring, serialization.
- **Business logic** — the wiring of lower boxes to produce the desired behaviour. Owns no transport and no storage.
- **Storage / integration** — database access, external service clients. Owns all query construction and driver concerns.

Each arrow between boxes is a contract.

## Rules

**Every box owns its own payload types.** The API request type is not the business-logic input. The database row type is not the API response. On day one they look identical; they always diverge. Sharing them welds the boxes together and kills independent evolution.

**Nothing leaks across an arrow.** Driver errors, SQL, ORM entities, and result sets stay inside the storage box. HTTP status codes, headers, request objects, and framework context types stay inside the exposure box. Business logic sees domain types only.

**Business logic is wiring.** If it constructs a query or reads a header, the responsibility is in the wrong box.

**Resist over-decomposition.** Separation follows responsibility, not DRY. A three-line helper does not need a box. A script does not need three layers. Apply this proportionally to what is being built.

## Error contracts

The failure path is an arrow like any other, and it is the one most often left uncontracted.

Each box publishes one error type. It carries a **kind** drawn from a small closed vocabulary — not-found, conflict, transient, invalid, unavailable — and the kind is what callers dispatch on. One type per box, not one per operation: per-operation types force a branch at every call site and make generic retry handling impossible.

Classify at the boundary. Vendor codes, driver errors, and transport status codes are translated into the kind vocabulary inside the box that owns the technology. Put that classification in a reusable helper, not in the error type's ancestry — sharing by inheritance puts the driver's type into your contract and breaks backend swaps.

Wrap deliberately. Three categories, three answers:

- **Operational failures** — unreachable store, write conflict, timeout, malformed query. Wrap and classify. These are why the contract exists.
- **Programming defects** — bad argument, broken invariant, null dereference. Propagate raw. Wrapping a bug as an operational failure invites retrying a deterministic crash and buries the origin.
- **Cancellation and environment signals** — shutdown, interruption, deadline from above, out of memory. Never wrap. They are not the box's to interpret, and swallowing cancellation breaks the caller's ability to stop work.

Unrecognized but caught defaults to non-retryable. Retrying into an unknown failure is the unsafe direction.

Preserve the cause. Keeping the underlying error reachable — cause chain, wrapped error, `source` — is not a leak: the kind is the contract surface, the cause is diagnostic. Callers may unwrap for logging or a rare backend-specific decision, never in ordinary control flow. Across a process boundary only the kind survives, so the vocabulary has to stand on its own.

Retry placement belongs in the contract. State whether the box retries internally before surfacing a transient failure. If the box and its caller both retry, attempts multiply.

## Physical boundaries

Boxes are always separated in code. The mechanism scales with the project; the separation does not.

One dial, cheapest to strongest:

- same module, separate packages or namespaces, with visibility keywords carrying the direction
- separate build modules, crates, or workspace packages in one repo
- separately versioned artifacts

Pick the lowest rung that holds. A module per box in a 900-line service is over-decomposition in physical form. One module for a service five teams touch has stopped enforcing anything.

What holds at every rung: exposure, business logic, and storage never share a package. Dependencies point one way — exposure → business logic → storage. A cycle means a responsibility is in the wrong box.

Know what your rung enforces. A Java package expresses direction but does not enforce it; anything on the classpath can import anything. Go packages and Rust crates enforce it at compile time. Where the rung doesn't enforce, add something that does — an ArchUnit test, a lint rule, a CI import check — so a violated arrow fails the build instead of waiting to be caught in review.

Move up a rung for a reason, not on principle: a second consumer, a different owning team, a different release cadence, tests slow enough to isolate, or direction violations that keep recurring.

The same dial applies inside a box. Contract and implementation separate by package in a small project, by module once a second implementation exists or a consumer shouldn't be compiling against the driver.

Ship the test double with the box. Consumers mock their neighbours by contract, so a fake buried in the owner's test sources is unreachable to them.

## When planning

Before proposing code for anything non-trivial, state the box breakdown. Keep it compact — a list or small ASCII diagram, not an essay. For each box:

- name and responsibility
- input type and output type
- what it depends on
- how it will be tested

Then state **build order**. Leaf boxes (storage, external clients) are built first because they have no dependencies. Contracts at the edges are *defined* first even though the boxes are built last, because they are what other teams and parallel workstreams mock against. Call out explicitly which items are parallelizable and which are blocked. Integration is the final step.

Where the work crosses a team or repo boundary, name the boundary and the contract that unblocks the other side.

## Test boundaries

Test each box against its own contract, and nothing more:

- **Storage** — test against a real or containerized store. This is the one layer where the real thing is worth the cost.
- **Business logic** — mock every box it wires. Test the wiring and the branching. Do not re-test storage behaviour; storage has its own tests that already prove it.
- **Exposure** — mock business logic. Test validation, auth wiring, serialization, and status mapping.
- **Integration** — tests the wiring of the whole product from the outside. It does not duplicate box tests.

Re-testing a neighbour's behaviour is the most common failure here. It doubles the cost of every change and couples tests that should be independent.

## Why this matters

The payoff is that a change lands in one box:

- change a tier limit → business logic and its test, nothing else
- add rate limiting → exposure box only
- swap the database backend → storage box only, no test changes upstream if the contract held
- add a gRPC entrypoint → reuse the business logic box as-is
- build a second product on the same data → reuse the storage box as-is

If a proposed change forces edits in a box that does not own the responsibility, the decomposition is wrong. Say so.
