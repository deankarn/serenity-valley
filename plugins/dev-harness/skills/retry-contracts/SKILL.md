---
name: retry-contracts
description: Design and audit retry behaviour through error contracts — errors carry retry advice (shouldRetry, retryAfter) via one shared interface, exactly one level retries per flow, and retries never fight an upstream time budget. Use when writing or reviewing retries, backoff, error handling for network, database, or queue calls, or any design with a failure path worth retrying. Invoke directly as /retry-contracts verify to audit a plan or codebase for retry violations.
when_to_use: Trigger when adding, changing, or reviewing retries, backoff, or jitter; when handling transient, timeout, 429, 503, or Retry-After failures; when writing an HTTP, gRPC, database, or queue client; when designing error types or error handling for a box; when building queue consumers or scheduled jobs; and whenever box-design is applied to a design with a failure path. Trigger on phrasings like "add retries", "retry on failure", "handle transient errors", "exponential backoff", "rate limited". Do not trigger for errors that are purely validation or for code with no remote or fallible dependency.
argument-hint: "[verify]"
arguments: [mode]
allowed-tools: Read Grep Glob
---

# Retry contracts

Mode: `$mode`

- Empty mode — apply the rules below to the current design or code. Do not run an audit.
- `verify` — read `${CLAUDE_SKILL_DIR}/references/verify.md` and run the audit it defines against whatever is in context. Audit only when the user asked for it. For a code audit, prefer delegating to the `dev-harness:design-auditor` agent when the Agent tool is available.

These rules extend the error contracts in `dev-harness:box-design`: each box publishes one error type, classified at the boundary. This skill covers what that error says about retrying, and who acts on it.

## Precedence

These rules are general. Project and organization guidance — an existing retry library, a mandated policy, framework conventions — wins on specifics. Apply these rules where that guidance is silent, and flag genuine conflicts to the user rather than resolving them silently.

## The retry interface

Define one retry interface — `Retryable` — as an interface, trait, or protocol that every box's error type implements. Exactly two methods:

- **`shouldRetry`** — whether the caller should retry. `should`, not `is`: the box combines protocol guarantees with documented behaviour and policy. An HTTP 500 is not retryable by definition, but many APIs document that callers should retry it; the box that owns that client decides, because real dependencies are not always ideal or within your control.
- **`retryAfter`** — optional duration before the next attempt. Absent means no opinion; the caller's strategy decides.

Idiomatic shapes: Go `ShouldRetry() bool` and `RetryAfter() (time.Duration, bool)`; Rust `fn should_retry(&self) -> bool` and `fn retry_after(&self) -> Option<Duration>`; Java `boolean shouldRetry()` and `Optional<Duration> retryAfter()`.

Define it once, centrally, as reusable code for the whole codebase or organization — one shared library per language — alongside the generic retry helper. The library has no dependencies of its own; every box depends on it, and it couples nothing to anything.

`Retryable` is the only retry signal. Callers never decide by inspecting status codes, driver codes, messages, or another box's error reasons. Reasons are for branching and status mapping, not retry. Inspecting another box's error also means importing its dependencies — a driver or client library pulled in only to read an error code — which is both leakage and bloat.

## Answering shouldRetry

- **Unknown is `false`.** A caught but unrecognized failure is not retried.
- **Programming defects and cancellation are never retried.** A defect stops the application where it is detected, and cancellation is never wrapped (see box-design's error contracts), so neither carries retry advice.
- **Deterministic failures answer `false`.** A rejected input or a duplicate write is wrapped, but retrying it fails the same way every time.
- **Idempotency is part of the answer.** A transient failure on a non-idempotent operation — a write that may have committed before the timeout — answers `false`, unless the operation carries an idempotency key that makes the retry safe. The box knows which of its operations are idempotent; that is why the box answers.
- **Wrapping re-answers.** A box that wraps another box's error implements the interface on its own error. It usually passes the inner answer through and may override it with knowledge the inner box lacks. Generic retry logic reads the outermost answer.

## Where to retry

**Retry at a level only if nothing above it imposes a limit the retry would fight.**

- Behind a request with a timeout — an API call, a gRPC call — no box inside the request retries. Every box surfaces retry advice, and the exposure box hands it to the caller, who owns the budget and the strategy.
- Inside a queue consumer, retrying in the handler fights poll, lease, visibility, or commit timeouts. Surface the failure and use the consumer's own retry mechanism — a retry topic, delayed redelivery, nack with delay — with `retryAfter` as the delay.
- In an unconstrained flow — a scheduled job with no deadline that nobody waits on — any one level may retry. Still exactly one per flow.

The owner of the time budget is usually the origination point: whoever started the flow and owns its deadline — an end-user client, a queue consumer, a scheduled job. A service handling a request is never the origination point; its caller is.

Compose retries at the call site with a generic helper rather than baking them into a box. The same box is reused under different budgets; a retry inside it fights every constrained caller.

## Never nest retries

Nested retries multiply: three levels of three attempts is 27 calls against a dependency that is already failing. Exactly one level retries per flow.

- **State it in the contract.** Each box declares whether it retries internally.
- **A box that retried owns the retry.** When it exhausts its attempts, its error answers `shouldRetry` with `false`. That is what stops the next level from retrying again.

Verify it: trace one transient failure from the bottom of the flow to the origination point and count the attempts.

## Crossing a process boundary

The underlying cause does not cross the wire. The error does, in the protocol's shape: its message, the reasons a caller can act on, and the retry advice.

- The exposure box translates advice into the protocol: HTTP 429/503 with `Retry-After`; gRPC status with `RetryInfo`; the broker's redelivery delay.
- The client box on the other side translates it back into its own error type implementing the same interface.

## The retry helper

Build it once, generic over the retry interface. Errors say whether and when; the caller supplies the strategy.

- Cap by attempts or elapsed time. Unbounded retries only for background work nobody waits on — never on a request path.
- Exponential backoff with jitter when `retryAfter` is absent; honour `retryAfter` when present.
- Respect cancellation and the deadline. If the next wait exceeds the remaining budget, give up immediately.
- When it gives up — attempts spent or budget exceeded — wrap the last error in one that answers `shouldRetry` with `false`, so no level above retries again. Errors that already said not to retry pass through unchanged.

## Testing

- Each box tests its own advice: a table of failure → `shouldRetry` and `retryAfter`, including idempotent vs non-idempotent operations and exhausted internal retries.
- The helper is tested once, against fake errors that implement the interface.
- Callers mock errors carrying advice. They never re-test a neighbour's classification.

## When planning

For each box on a failure path, state: its error reasons, how it answers `shouldRetry` (including idempotency), whether it provides `retryAfter`, and whether it retries internally. Then name the one level that retries in each flow and why its budget allows it.

When diagramming a retry flow, use a sequence diagram that shows the advice travelling up to the retrying level — Mermaid in rendered documents, a small ASCII diagram or list in terminal replies.
