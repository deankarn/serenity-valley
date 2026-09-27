# Verify

Pick the checklist by what is in context. A proposed plan, design doc, or diagram → **Plan audit**. Source files, a diff, or a repo → **Code audit**. Both present → run both, plan first.

State which one you ran. Report findings grouped by flow, then by box, each with a severity and the specific location. Explicitly list what is already correct and should be left alone.

Severity:
- **amplifies** — retries multiply or fight a time budget: nested retries, retrying inside a constrained request or consumer, unbounded retries on a request path. Fix before merging.
- **unsafe** — a retry can do harm: non-idempotent operation retried, defect or cancellation retried, unknown failure retried.
- **leak** — a caller decides retry from another box's internals instead of the retry interface.
- **note** — smell, judgement call, or something to watch.

Give each finding a one-line fix direction. Do not rewrite code during an audit unless asked. Report, then offer.

---

## Plan audit

1. Does every box on a failure path name its error type and how it answers `shouldRetry` and `retryAfter`?
2. Is there one shared `Retryable` interface and retry helper, defined centrally per language, rather than per-box or per-call retry logic?
3. For each flow, is the one retrying level named, with the time budget that allows it? Flag any flow with zero or more than one.
4. Does any box retry inside a constrained flow — an API request, an RPC, a queue handler with poll or commit timeouts?
5. Does each box declare whether it retries internally, and does an exhausted internal retry surface `shouldRetry = false`?
6. Are non-idempotent operations identified, and do they either answer `false` on ambiguous failures or carry an idempotency key?
7. Where the flow crosses a process boundary, is the wire translation stated in both directions?
8. Is the retry strategy (cap, backoff, jitter, deadline handling) owned by the retrying caller, not by the errors?

---

## Code audit

Use Grep and Glob. Adapt the patterns to the language — these are signals, not literal searches.

**Nesting and placement**
- Retry loops, retry libraries, or backoff calls at more than one level of the same call path. Trace one transient failure end to end and count the attempts.
- Retries inside request handlers, service methods called from handlers, or queue message handlers.
- `sleep` or backoff inside a queue handler instead of redelivery with a delay.
- A retry baked into a reusable box rather than composed at the call site.
- An internal retry that, once exhausted, still returns an error that says to retry.

**Answering**
- Error types that do not implement the shared retry interface.
- Callers switching on HTTP status, driver codes, error messages, or another box's error reasons to decide whether to retry.
- A caller importing a driver or client library only to inspect an error.
- Unrecognized failures, defects, or cancellation answering `shouldRetry = true`.
- Retried writes with no idempotency key and no idempotency reasoning.
- A wrapping error type that drops the inner advice instead of passing it through or deliberately overriding it.

**Retry helper**
- More than one retry helper, or retry logic duplicated inline.
- No cap, or unbounded retries on a request path.
- No jitter when `retryAfter` is absent; `retryAfter` ignored when present.
- Retry loop that ignores cancellation or the deadline, or sleeps past the remaining budget.
- A helper that gives up but returns an error still answering `shouldRetry = true`.

**Wire translation**
- An exposure box that returns retryable failures with no `Retry-After`, `RetryInfo`, or equivalent.
- A client box that ignores `Retry-After` or retryable status codes when building its error.

**Tests**
- Boxes with no test of their retry advice.
- The retry helper re-tested in every caller.
- Caller tests that depend on a neighbour's real classification rather than a mocked error carrying advice.
