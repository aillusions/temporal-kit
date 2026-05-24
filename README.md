# temporal-kit

Reusable building blocks for Temporal workflow code — sagas, approval gates, polling, FSMs, and other patterns that are too fiddly to rewrite per workflow. TypeScript today, Go planned.

## Status

Early. TypeScript only. APIs will break. Not yet published.

## Install

> Not published yet. Once available:

```sh
npm install @temporal-kit/patterns @temporalio/workflow
```

`@temporalio/workflow` is a peer dependency — choose your own version.

## Quick example (forward-looking)

A saga with compensations. **Illustrative — the implementation does not exist yet.**

```ts
import { saga } from '@temporal-kit/patterns/saga';
import { reserveInventory, chargeCard, shipOrder, refund, releaseInventory } from './activities';

export async function placeOrderWorkflow(order: Order): Promise<void> {
  await saga()
    .step({
      do:   () => reserveInventory(order),
      undo: () => releaseInventory(order),
    })
    .step({
      do:   () => chargeCard(order),
      undo: () => refund(order),
    })
    .step({
      do:   () => shipOrder(order),
    })
    .run();
}
```

If any step throws, prior `undo`s run in reverse order. See `docs/patterns/saga.md` for the spec.

## Repo layout

```
packages/ts/
  core/         # shared primitives
  patterns/     # saga, approval gate, pollUntil, FSM, ...
  testing/      # replay test helpers, fixture loaders
  examples/     # runnable workflows
packages/go/    # placeholder

docs/
  patterns/     # language-agnostic specs — source of truth
  adr/          # architecture decisions
```

Browse [`docs/patterns/`](./docs/patterns/) for the catalog of primitives and their specs.

## Replay safety

Every primitive runs inside Temporal's deterministic replay. Behavior changes can break in-flight workflows. Each primitive's README declares whether it is **safe to upgrade mid-flight** or **requires draining**. Every PR touching a primitive ships replay tests against committed history fixtures.

Full policy in [CLAUDE.md](./CLAUDE.md).

## Contributing

Contribution guide TBD. In the meantime: specs land in `docs/patterns/` before code, and replay fixtures are not optional. See [CLAUDE.md](./CLAUDE.md) for the working rules.
