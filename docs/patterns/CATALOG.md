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

### child-2pc
Inter-workflow two-phase commit: coordinate prepare-then-commit across multiple child workflows. Abort all on any prepare failure. (Previously listed as `two-phase-commit`; renamed to disambiguate from intra-workflow `commit-after-prep`.)
Status: planned

### commit-after-prep
Intra-workflow two-phase: a cancelable preparation phase followed by a non-cancellable commit. Phase 1 may be interrupted or retried; phase 2 runs to completion under a non-cancellable scope. Distinct from `child-2pc` (one workflow vs. many).
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
Collect signals into a buffer; flush after a debounce window. Re-schedules the flush if new signals arrive during the window; drains the remaining buffer on explicit close. Acts on the aggregate, not each signal individually.
Status: planned

### deadliner
Deadline tracker with `postpone()`, `expired()`, and `waitNextTick()`. Small primitive used standalone or as a building block by other deadline-aware primitives (`idle-timeout`, `lease`, `poll-until`).
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
Accumulate items from one or more precursor sources; flush as a batch when size N, time T, or an external trigger fires. Supports mid-batch appends and coordinated flush across multiple precursors.
Status: planned

---

## Concurrency and rate limiting

### workflow-mutex
Serialize async operations inside a single workflow to prevent concurrent state updates across signal / update / query handlers. Configurable error-propagation modes (break-chain vs. ignore-and-continue). Foundational — many other primitives use this internally.
Status: planned

### interruptible-scope
Cancellation-scope wrapper with an external `interrupt()` trigger and a pollable `wasInterrupted()` predicate available inside the callback. Used directly when a workflow needs externally-cancellable inner work distinct from workflow cancel, and as a building block for `commit-after-prep` and `singleton-child`.
Status: planned

### update-tracker
Counts in-flight async update handlers. Used standalone to gate shutdown ("are there updates still running?") and as a building block for `idle-timeout`'s grace period.
Status: planned

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

### singleton-child
At-most-one child workflow at a time, with enable/disable toggle and graceful cancel on shutdown. Uses `getExternalWorkflowHandle().cancel()` for graceful cancellation rather than parent-close; ships an upgrade path via `patched()` for backward-compatible behavior changes.
Status: planned

### idle-timeout
Extend workflow lifetime by tracking activity and resetting an idle deadline on each bump. Terminal-state guard prevents bumping after shutdown; configurable grace period lets in-flight updates settle before close.
Status: planned

---

## Fire-and-forget

### detached-task
Register fire-and-forget async operations or child workflows; track them as a set, optionally cancel all in-flight, and wait for everything to settle. Two variants planned:
- bare promise / async callback (lightest)
- structured `commit-after-prep` (phase 1 + phase 2) detached

Child variants use `ParentClosePolicy.ABANDON` so the parent can close while children continue.
Status: planned

---

## Observability

### marker
Debug local-activity for recording structured workflow events. Disabled by default, enabled per-component. Used inline by workflow authors to emit trace points, and by other primitives to expose their internal state-machine transitions to operators.
Status: planned

---

## Suggested first three

These have the best ratio of value to scope-clarity and will shake out the library's API conventions:

1. `saga` — exercises compensation, error handling, replay-safety policy
2. `approval-gate` — exercises signals, timers, typed channels
3. `poll-until` — exercises activity invocation defaults, cancellation, deadline tracking

Build these end-to-end (spec → impl → tests → fixtures → README) before scaffolding any others. The conventions discovered here become the template for the rest.

`workflow-mutex` is a strong fourth candidate because many later primitives depend on it.

---

## Out of scope (explicitly not in this catalog)

- Activity implementations
- Worker setup, task queue management, infra
- Temporal SDK wrappers that don't add semantic value
- Generic retry/backoff utilities — that's SDK territory
- Testing infrastructure beyond the `marker` primitive (lives in `packages/ts/testing`, separate concern)
- Private helpers inside a primitive's source that are not exported — those live by doc-comment, no spec needed
