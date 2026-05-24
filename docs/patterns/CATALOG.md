# Pattern Catalog

The full list of primitives planned for temporal-kit. Each entry gets its own spec doc (`docs/patterns/<name>.md`) before implementation. This file is the **single source of truth for pattern status**. For what is in motion right now, see [`../PROGRESS.md`](../PROGRESS.md).

Status legend:
- `planned` — agreed, not started
- `spec` — spec doc in progress or written
- `impl` — implementation in progress
- `released` — shipped in a published version
- `deferred` — agreed concept, postponed

---

## Long-running coordination

### saga
Forward steps with paired compensations. On failure, run compensations in reverse order. Workflow body becomes `saga([step1, step2, step3])`.
Status: spec — see [`saga.md`](saga.md)

### two-phase-commit
Coordinate prepare-then-commit across multiple child workflows. Abort all on any prepare failure.
Status: planned

### approval-gate
Wait for signal-based approval with timeout. Configurable auto-reject or auto-approve on timeout. Returns typed decision.
Status: planned

### multi-approver
Quorum, unanimous, or first-response approval across N parties. Handles partial results and timeouts.
Status: planned

---

## Polling and waiting

### poll-until
`pollUntil(activity, predicate, { interval, maxDuration, backoff })`. The most-reached-for primitive.
Status: planned

### wait-for-external-state
Poll external system to reach state X, with built-in escalation: poll faster → alert → give up.
Status: planned

### debounced-signal-handler
Collect signals over a window, act on the aggregate rather than each one.
Status: planned

---

## Human-in-the-loop

### review-workflow
Send notification, wait for response, reminder cadence, escalation chain, final timeout decision.
Status: planned

---

## Scheduling and recurrence

### recurring-task
"Do X every N, until condition Y." Handles continue-as-new automatically.
Status: planned

### scheduled-batch
Collect items via signals until time T or size N, then process as a batch.
Status: planned

---

## Rate limiting and concurrency

### token-bucket
Token-bucket rate limiter backed by activity calls, for workflows driving external APIs with quotas.
Status: planned

### concurrency-limiter
Bounded fan-out: "run these 1000 child workflows, max 10 in flight."
Status: planned

---

## Idempotent external effects

### exactly-once-call
Wrapper that derives an idempotency key, calls the activity, records the result in workflow state. Safe to re-enter.
Status: planned

### outbox
Stage effects in workflow state, flush via single activity, mark flushed.
Status: planned

---

## State machines

### fsm
Typed finite state machine: declare states, transitions, signal and timer triggers. Replaces hand-rolled `while(true) { switch(state) }` loops.
Status: planned

### event-sourced-aggregate
Apply events to state, expose queries. For workflows that model a long-lived aggregate.
Status: planned

---

## Negotiation and multi-party

### auction
Open window, collect bids via signals, close, resolve by rule.
Status: planned

### consensus
Gather votes from N child workflows or signals, decide by rule (majority, unanimous, weighted).
Status: planned

---

## Money and quantities

### reservation
Reserve → confirm-or-release with timeout. Compensations wired in.
Status: planned

### ledger-entry
Debit-credit with rollback. Idempotent.
Status: planned

---

## Lifecycle

### lease
Workflow holds a lease, renews on schedule, releases on cancel or expiry.
Status: planned

### subscription
Start, periodic charge child workflows, pause/resume signals, cancel with prorating.
Status: planned

---

## Suggested first three

These have the best ratio of value to scope-clarity and will shake out the library's API conventions:

1. `saga` — exercises compensation, error handling, replay-safety policy
2. `approval-gate` — exercises signals, timers, typed channels
3. `poll-until` — exercises activity invocation defaults, cancellation, deadline tracking

Build these end-to-end (spec → impl → tests → fixtures → README) before scaffolding any others. The conventions discovered here become the template for the rest.

---

## Out of scope (explicitly not in this catalog)

- Activity implementations
- Worker setup, task queue management, infra
- Temporal SDK wrappers that don't add semantic value
- Generic retry/backoff utilities — that's SDK territory
- Testing infrastructure (lives in `packages/ts/testing`, separate concern)