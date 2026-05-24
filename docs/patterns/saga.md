# saga

Spec for the `saga` primitive: forward steps with paired compensations, automatic rollback on failure.

**Status:** draft. No implementation yet. This document is the source of truth; the TypeScript and (future) Go implementations must satisfy what is written here.

---

## 1. Concept

A saga is a sequence of side-effectful steps where each step has an associated compensation. If any step fails partway through, the saga runs compensations for the steps that already succeeded, in reverse order.

Compared to hand-rolled `try { stepA(); stepB(); stepC() } catch { undoB(); undoA() }` patterns, the saga primitive owns the bookkeeping: which forward steps ran, which compensations are pending, what to do when a compensation itself fails, how cancellation interacts with rollback, how all of this survives replay.

Reach for it whenever a workflow has more than two side-effects that must be unwound on failure. Two or fewer: hand-rolling is fine and arguably clearer.

---

## 2. States

```
                ┌──────────┐
                │ pending  │  run() not yet called
                └────┬─────┘
                     │ run()
                     ▼
                ┌──────────┐    forward step fails / workflow cancel
                │ running  │ ────────────────────────────────┐
                └────┬─────┘                                 │
                     │ all forward steps succeed             ▼
                     │                              ┌──────────────┐
                     ▼                              │ compensating │
              ┌────────────┐                        └──────┬───────┘
              │ completed  │ (terminal)                    │
              └────────────┘                               ├─ all comps ok      ─▶ compensated
                                                           ├─ some comp threw   ─▶ compensation-failed
                                                           └─ nothing to comp   ─▶ failed
```

State definitions:

- **pending** — handle constructed, `run()` not yet invoked. No history events recorded.
- **running** — `run()` invoked, forward steps executing sequentially. Exits to `completed` (last forward step succeeded), `compensating` (a step failed or cancel arrived with ≥1 successful step), or `failed` (the first step failed and no compensations are runnable).
- **compensating** — at least one forward step succeeded; compensations are executing in reverse order under a non-cancellable scope.
- **completed** *(terminal)* — every forward step succeeded; zero compensations ran.
- **compensated** *(terminal)* — at least one forward step failed (or workflow cancelled); every triggered compensation succeeded.
- **compensation-failed** *(terminal)* — at least one compensation threw. Some others may still have succeeded; full outcome is reported per-step.
- **failed** *(terminal)* — saga terminated without compensating: either step 0 failed before any state existed, or a step opted out of compensation via `classify: 'fail-fast'` (§7).

Terminal state is always observable via the `SagaOutcome` returned by `run()` (§3).

---

## 3. API surface

Proposed TypeScript signatures. Subject to refinement before implementation; treat anything marked *(tentative)* as up for change.

### Step shape

```ts
interface SagaStep<R = unknown> {
  /** Unique within the saga. Recorded in workflow history; treat as immutable. */
  name: string;

  /** Side-effectful work. Must be an activity call or child workflow (§4). */
  forward: () => Promise<R>;

  /** Inverse of `forward`. Omit if the step is purely informational. */
  compensate?: (ctx: CompensateContext<R>) => Promise<void>;

  options?: SagaStepOptions;
}

type CompensateContext<R> =
  | { phase: 'after-success'; value: R }
  | { phase: 'after-failure'; error: unknown };

interface SagaStepOptions {
  /**
   * If true AND `forward` throws, call `compensate` with
   * `{ phase: 'after-failure', error }`. Default: false.
   *
   * Use when partial work may have escaped before the throw
   * (e.g., a network call that timed out but possibly committed).
   */
  compensateOnForwardFailure?: boolean;

  /**
   * Classify a forward error. Default rules in §7.
   *   'compensate' — trigger rollback.
   *   'fail-fast'  — terminal status = 'failed', no compensations run.
   */
  classify?: (err: unknown) => 'compensate' | 'fail-fast';
}
```

### Constructor

```ts
function saga(steps: SagaStep[], options?: SagaOptions): SagaHandle;

interface SagaOptions {
  /**
   * What to do when a compensation throws. Default: 'continue'.
   *   'continue' — run remaining compensations, collect all errors.
   *   'abort'    — stop, leave remaining compensations un-run.
   */
  onCompensationFailure?: 'continue' | 'abort';

  /** Register a workflow query named `<name>` exposing current state. Off by default. */
  exposeQuery?: string;  // tentative — see open question 2
}

interface SagaHandle {
  run(): Promise<SagaOutcome>;
}
```

### Fluent sugar (optional)

```ts
saga()
  .step({ name: 'reserve', forward: reserveInventory, compensate: releaseInventory })
  .step({ name: 'charge',  forward: chargeCard,       compensate: refund })
  .step({ name: 'ship',    forward: shipOrder })
  .run();
```

The array form is canonical. Implementations MAY ship the builder; it MUST be exactly equivalent to the array form.

### Return type

```ts
type SagaStatus =
  | 'completed'
  | 'compensated'
  | 'compensation-failed'
  | 'failed';

interface SagaOutcome {
  status: SagaStatus;

  /** Forward step results keyed by step name. Only includes steps that succeeded. */
  results: Record<string, unknown>;

  /** Set when status !== 'completed'. The step whose forward triggered termination. */
  failedStep?: {
    name: string;
    error: unknown;
  };

  /** One entry per compensation that was triggered. Order = reverse forward order. */
  compensations: Array<{
    stepName: string;
    status: 'ok' | 'failed' | 'skipped';
    error?: unknown;
  }>;
}
```

`run()` resolves on every terminal state — including failure. It does NOT reject.

Rationale: making `run()` reject would couple saga's error semantics to the caller's `try/catch` shape and obscure the structured outcome. Callers who want throw-on-failure can write a one-liner:

```ts
const out = await saga(steps).run();
if (out.status !== 'completed') throw new SagaFailedError(out);
```

---

## 4. Guarantees

### Ordering

- Forward steps run **strictly sequentially** in declaration order. No implicit parallelism.
- Compensations run **in reverse order** of the forward steps that succeeded.
- The step that failed mid-forward is **not compensated by default**. Opt in via `compensateOnForwardFailure: true` per step.

Rationale: most failed forward steps have done nothing observable — they threw before completing. Compensating them is at best a no-op, at worst destructive (e.g., refunding a charge that never happened double-credits). Default to safe; let the step author opt in when partial-work is possible.

### On compensation failure

- Default (`onCompensationFailure: 'continue'`): the saga runs every remaining compensation, collects every error, and reports each in `outcome.compensations`. Terminal state is `compensation-failed`.
- `'abort'`: stop on first failed compensation. Remaining compensations are reported with `status: 'skipped'`.

Rationale: a failed compensation usually means external state is already broken. The best remaining move is to attempt the rest of the rollback so the operator has a complete picture. `'abort'` exists for cases where a failed compensation invalidates the next.

### Idempotency

User-supplied `forward` and `compensate` MUST be idempotent under retry. The saga primitive does not deduplicate; Temporal's activity retry mechanism may invoke either function multiple times before the saga observes a terminal outcome.

This is a contract the saga relies on but does not enforce. Violating it is a user bug.

### Determinism

`forward` and `compensate` MUST be one of:

- A Temporal activity call (e.g., `proxyActivities`).
- A child workflow invocation (`executeChild`, `startChild`).
- Pure deterministic workflow code that wraps the above (e.g., a small function that picks an activity based on a deterministic input).

They MUST NOT:

- Read non-deterministic data (`Date.now()`, `Math.random()`).
- Register signal or query handlers.
- `await` on anything other than Temporal-provided primitives.

Violating determinism poisons replay for the entire workflow, not just the saga.

---

## 5. Replay-safety

A saga's behavior is reconstructed from history on every replay. Any change to the saga definition that alters which activities are invoked, or in what order, risks a non-deterministic replay against in-flight workflows.

### Step identity

**Proposal:** identity is by `name`, sequencing is by index.

- `name` is the stable identifier recorded in `outcome.compensations`, in logs, and in any marker/heartbeat the implementation records to history.
- Index determines execution order but is not used for replay matching.

Tradeoff: name-based identity lets step bodies evolve while history matching still works. Index-based identity is simpler to implement but makes any reorder unsafe even when semantics are preserved. We accept the implementation cost to keep migration paths open.

Names MUST be unique within a saga. The implementation SHOULD enforce uniqueness at construction time.

### Safe in-flight changes

Safe for sagas already mid-flight:

- Renaming internal local variables in `forward` / `compensate` bodies.
- Changing the body of `forward` or `compensate` PROVIDED the underlying activity name and signature in Temporal are unchanged. (The saga sees activity invocations; the workflow code around them re-executes deterministically.)
- Adjusting per-step `options` for steps that have not yet executed in the current run — but see *borderline* below for `compensateOnForwardFailure`.

### Unsafe in-flight changes

Require a major version bump and a drain of in-flight workflows:

- Reordering steps.
- Removing a step.
- Renaming a step (the old name may appear in already-recorded history).
- Inserting a step before any step that has already executed in any in-flight workflow.
- Changing a step's `forward` from activity-A to activity-B (changes the recorded activity name).

### Borderline

- **Appending a step at the end.** Safe ONLY if no in-flight workflow has yet reached the end of the saga. In general, treat as unsafe: recommend a major bump unless you can prove zero in-flight saga completions overlap the deploy window.
- **Toggling `compensateOnForwardFailure`.** Safe to flip OFF → ON for steps that have not yet failed. Flipping ON → OFF is unsafe: replay may have already recorded a compensation invocation that the new code path would not produce.
- **Changing `onCompensationFailure`.** Safe for any saga that has not yet entered the `compensating` state.

When in doubt: bump the major and drain.

---

## 6. Cancellation

### During forward phase

- Workflow cancellation interrupts the in-flight forward step per Temporal's standard activity-cancellation rules.
- The interrupted step is treated as failed; its own compensation runs only if `compensateOnForwardFailure: true`.
- Compensations for previously completed steps run in reverse order.
- Terminal state: `compensated` or `compensation-failed`.

### During compensation phase

- Workflow cancellation does NOT abort the compensation phase.
- Compensations run inside a **detached cancellation scope** (in the TS SDK: `CancellationScope.nonCancellable`).
- Compensation functions MUST be written to tolerate this: they complete or fail by their own logic and MUST NOT short-circuit on cancellation.
- Terminal state is determined by compensation outcomes as usual.

Rationale: a cancel arriving mid-rollback is the worst possible time to stop rolling back. We prefer to leave external state consistent over honoring the cancel promptly.

### Cancel before run()

- `run()` resolves with `{ status: 'failed', results: {}, compensations: [], failedStep: undefined }`.
- No forward step ran; no compensation needed.

### Per-step cancellation scopes

- Each forward step runs in its own cancellation scope so that per-step timeouts (open question 7) can cancel one step without cascading to siblings.
- The compensation phase uses a single non-cancellable scope wrapping all compensations.

---

## 7. Error classification

### Defaults

| Thrown by step | Default action |
|---|---|
| `ApplicationFailure` (activity terminal error: retries exhausted or non-retryable) | compensate |
| `CancelledFailure` (workflow cancel propagated to step) | compensate per §6 |
| `ChildWorkflowFailure` terminal | compensate |
| Anything else (raw `Error`, custom thrown values) | propagate to workflow (treat as user bug) |

### Interaction with activity retries

The saga only observes terminal outcomes. Activity retry policies (`retryPolicy`, `nonRetryableErrorTypes`) are configured on the activity, not on the saga. The saga sees one of:

- success
- retryable-error-exhausted → `ApplicationFailure` → compensate
- non-retryable → `ApplicationFailure` → compensate
- cancel → `CancelledFailure` → compensate per §6

### Per-step override

```ts
{
  name: 'charge',
  forward: chargeCard,
  compensate: refund,
  options: {
    classify: (err) =>
      err instanceof InsufficientFundsError ? 'fail-fast' : 'compensate',
  },
}
```

- `'compensate'` — trigger rollback (default behavior).
- `'fail-fast'` — saga transitions directly to terminal `failed`. No compensations run. The error is captured in `outcome.failedStep.error`.

Rationale for `fail-fast`: some errors are business-level decisions that compensation cannot fix (e.g., "user not eligible" — there is nothing to undo because the failing step represents a denied operation, not a partial commit). Failing fast preserves the prior side-effects, which is the desired outcome.

---

## 8. Examples

Illustrative TypeScript. `@signus/temporal-kit-patterns/saga` does not yet exist; these snippets describe the intended API shape.

### Happy path

```ts
import { saga } from '@signus/temporal-kit-patterns/saga';
import {
  reserveInventory, chargeCard, shipOrder,
  releaseInventory, refund,
} from './activities';

export async function placeOrderWorkflow(order: Order) {
  const outcome = await saga([
    {
      name: 'reserve',
      forward:    () => reserveInventory(order),
      compensate: () => releaseInventory(order),
    },
    {
      name: 'charge',
      forward:    () => chargeCard(order),
      compensate: () => refund(order),
    },
    {
      name: 'ship',
      forward:    () => shipOrder(order),
    },
  ]).run();

  // outcome = {
  //   status: 'completed',
  //   results: { reserve: {...}, charge: {...}, ship: {...} },
  //   compensations: [],
  // }

  if (outcome.status !== 'completed') {
    throw new Error(`order failed at ${outcome.failedStep?.name}`);
  }
}
```

### Mid-saga failure

```ts
// chargeCard throws ApplicationFailure (card declined, retries exhausted).
// 'reserve' already succeeded → releaseInventory runs.
// 'ship' never starts.

const outcome = await saga([
  { name: 'reserve', forward: () => reserveInventory(order), compensate: () => releaseInventory(order) },
  { name: 'charge',  forward: () => chargeCard(order),       compensate: () => refund(order) },
  { name: 'ship',    forward: () => shipOrder(order) },
]).run();

// outcome = {
//   status: 'compensated',
//   results: { reserve: { reservationId: 'r-123' } },
//   failedStep: { name: 'charge', error: <ApplicationFailure: 'card declined'> },
//   compensations: [
//     { stepName: 'reserve', status: 'ok' },
//   ],
// }
```

### Cancellation after step 2

```ts
// Workflow is cancelled after 'charge' completes, before 'ship' starts.
// Compensations run for 'charge' then 'reserve' under a non-cancellable scope.

// outcome = {
//   status: 'compensated',
//   results: { reserve: {...}, charge: {...} },
//   failedStep: undefined,   // see open question 9 (cancel attribution)
//   compensations: [
//     { stepName: 'charge',  status: 'ok' },
//     { stepName: 'reserve', status: 'ok' },
//   ],
// }
```

---

## 9. Open questions

Resolve before tagging v1.0.0. Each is left explicit so future-us doesn't backfill a decision silently.

1. **Parallel step groups.** Should `saga([...])` accept nested arrays representing parallel-within-sequential? E.g., `[stepA, [stepB1, stepB2], stepC]`. Risk: complicates the state machine and compensation-order semantics significantly. *Lean:* no for v1; revisit when a concrete use case appears.

2. **Query handler for current state.** Should the saga register a workflow query exposing current saga state (which step is running, which compensations have run)? Useful for operator debugging. Requires the saga to register a handler in the workflow context, which couples it to `@temporalio/workflow`'s `setHandler`. *Lean:* yes, opt-in via `exposeQuery: 'sagaState'`.

3. **Passing step results forward.** Should later steps' `forward` receive earlier steps' results as an argument? E.g., `forward: (prev) => ship(prev.reserve.reservationId)`. Replay-safe in principle but ties step bodies to step names, making renames riskier and the type signatures awkward. Alternative: closures over workflow-local variables (the current pattern). *Lean:* defer; closures are fine.

4. **Testing strategy in `@signus/temporal-kit-testing`.** Compensations are exercised via replay against committed histories. Open: do we need a synthetic fixture generator (faster, less realistic) in addition to recorded histories? *Lean:* recorded fixtures for replay tests; synthetic state-machine tests for the saga's own internals.

5. **Name uniqueness enforcement.** Runtime (throw on construction) or documentation only? *Lean:* throw on construction. Cheap check, catches a class of subtle replay bugs early.

6. **Idempotency keys.** Should the saga auto-derive an idempotency key per step (e.g., `${workflowId}:${stepName}`) and pass it to activity calls? Useful, but couples saga to a specific activity contract. *Lean:* no; leave to the activity author. May revisit if a pattern emerges.

7. **Per-step timeout.** `options.timeout` to cancel a forward step after duration X without cancelling the whole saga. Mechanism is clear (per-step cancellation scope). *Lean:* yes, but defer to v1.1 unless a downstream primitive needs it.

8. **Continue-as-new.** For sagas with many steps, history may exceed Temporal's limits. Does the saga have built-in continue-as-new support, or is that the caller's responsibility? *Lean:* caller's responsibility; sagas with >10 steps are ergonomically out of scope already.

9. **Cancel attribution.** When a cancel arrives between steps (no specific forward step in flight), what populates `outcome.failedStep`? Options: (a) the next step that would have run, with a synthesized `CancelledFailure`; (b) `undefined`, with a separate `cancelledAt: { afterStep: string }` field. *Lean:* (b). Cleaner — failedStep means "this step's forward threw."

10. **Outcome serialization.** Should `SagaOutcome` be JSON-serializable so it can be exposed via the query handler (open question 2) and stored as a marker in history? This constrains `results` and `error` to be JSON-serializable, which is already a Temporal expectation. *Lean:* yes; document the constraint.

11. **Compensation re-entry on workflow continue-as-new.** If a saga is mid-compensation when the workflow continues-as-new (manually triggered by user code), what's the contract? *Lean:* saga refuses to continue-as-new mid-compensation; document that this is the caller's responsibility to avoid.

---

## 10. Out of scope

- **Distributed sagas across Temporal namespaces.** A saga lives in one workflow.
- **Non-Temporal coordination.** No fallback to direct HTTP, no DB transactions, no message queues. Forward and compensate go through activities or child workflows.
- **Custom retry policies set by the saga.** Retry is the activity's concern. The saga sees terminal outcomes only.
- **Compensation SLAs / deadlines.** No "compensation must complete within X minutes or escalate." That belongs in a different primitive.
- **Audit log emission to external sinks.** Logging is the application's responsibility. The saga MAY emit structured events for the query handler (open question 2) but does not push anywhere.
- **GUIs / dashboards.** Outside the scope of a workflow primitive.
- **Auto-generated compensations.** No magic: the user supplies the inverse explicitly.
