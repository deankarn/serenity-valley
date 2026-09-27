# Verify

Pick the checklist by what is in context. A proposed plan, design doc, or diagram → **Plan audit**. Source files, a diff, or a repo → **Code audit**. Both present → run both, plan first.

State which one you ran. Report findings grouped by box, each with a severity and the specific location. Explicitly list what is already correct and should be left alone — an audit that only produces complaints gets ignored.

Severity:
- **breaks-composability** — a change in one box will force a change in another. Fix before merging.
- **leak** — a type or concern crossed an arrow it should not have.
- **test-boundary** — a box is testing something a neighbour already proves, or not testing what it owns.
- **note** — smell, judgement call, or something to watch.

Give each finding a one-line fix direction. Do not rewrite code during an audit unless asked. Report, then offer.

---

## Plan audit

1. Is every box a responsibility, or is it a grouping of files? Boxes justified by size are not boxes.
2. Does each box declare its own input and output types? Flag any type named as both an API payload and a business-logic or storage payload.
3. Is business logic free of transport and storage concerns? Look for a plan step that has the logic layer building a query or reading a header.
4. Are the edge contracts defined before the work that depends on them? A plan that builds the exposure layer before naming its contract blocks everyone downstream.
5. Is build order leaf-first, with parallelizable and blocked items called out separately, and do milestones follow the blocked-by edges?
6. Is there a stated test strategy per box, with neighbours mocked?
7. Over-decomposition: does the box count match the size of the problem? Say so if it does not.
8. Cross-team or cross-repo edges: is each one named, with the contract that unblocks the other side?
9. Does each box name its error type and the reasons it can surface? A plan silent on the failure path has an uncontracted arrow. Flag absence (not found) modelled as an error rather than an empty collection or optional.
10. Retry behaviour: unless it has already run, invoke the `dev-harness:retry-contracts` skill with `verify` and run its plan audit. Report its findings in a separate section.
11. Does the plan say which rung each boundary sits on (packages within a module, separate modules, versioned artifacts), and does the rung match the project's size?
12. Where the language does not enforce direction, is there a stated enforcement mechanism? Flag a Java or TypeScript plan that relies on package layout alone.

---

## Code audit

Use Grep and Glob. Adapt the patterns to the language in the repo — these are signals, not literal searches.

**Type leakage across arrows**
- Storage types in exposure signatures or responses: ORM entities, `Row`, `Record`, `Entity`, `Model`, nullable wrapper types, result-set types.
- Transport types inside business logic: request/response objects, framework context types, status codes, headers, cookies, multipart types.
- A single type serving as request payload, domain object, and persisted row.

**Misplaced responsibility**
- Query construction, SQL strings, or ORM calls inside a handler or controller.
- Business branching inside a handler beyond validation and status mapping.
- Transaction management spread across more than one layer.
- Auth decisions duplicated in both exposure and business logic.

**Error contracts**
- Driver, ORM, or HTTP client errors surfaced verbatim past their owning box.
- A caller importing a driver or client library only to inspect an error.
- Absence (not found) raised or returned as an error instead of an empty collection or optional.
- One box's error type imported by a box two arrows away.
- An error type whose ancestry includes a driver or framework type — the leak arrives through the supertype.
- One error type per operation rather than one per box. Callers end up with a catch block per call site.
- No catch-all on the box's public entry points, so unchecked or unexpected failures escape unclassified.
- Cancellation, interruption, shutdown, or out-of-memory caught and wrapped as an operational failure.
- Programming defects — malformed query, broken invariant, null dereference — wrapped, returned as errors, or classified as retryable instead of stopping the application where they are detected.
- Operational failures panicking or crashing the application. Stopping is reserved for defects where the application cannot function.
- Cause chain dropped on wrap, or unwrapped in ordinary control flow rather than for logging.
- Retry behaviour: unless it has already run, invoke the `dev-harness:retry-contracts` skill with `verify` and run its code audit. Report its findings in a separate section.

**Physical boundaries**
- Exposure, business logic, and storage sharing a package or namespace.
- Dependency cycles between boxes, or any import pointing against exposure → business logic → storage.
- A language that does not enforce direction with no compensating check: no ArchUnit test, lint rule, or CI import assertion.
- Consumers compiling against a driver or client library because the box ships contract and implementation together.
- Rung mismatch in either direction: a module per box in a small service, or one module for something several teams touch.
- A box's test double living only in its own test sources, so no consumer can reach it.

**Test boundaries**
- Business-logic tests that connect to a real store, spin containers, or import storage internals.
- Storage tests that mock the store entirely — they prove nothing.
- Exposure tests that assert business rules already asserted in the logic layer.
- The same behaviour asserted at two or more layers.
- A box with no test that owns real behaviour.

**Composability probe**

Pick two or three plausible near-term changes for this codebase — a limit or policy change, a new entrypoint, a backend swap — and trace which files each would touch. If a change fans out beyond the box that owns the responsibility, that fan-out is the finding. Name the files.
