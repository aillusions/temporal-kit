# @signus/temporal-kit-patterns

Workflow patterns for Temporal: sagas, approval gates, `pollUntil`, FSMs, and other building blocks.

## Status

Empty placeholder. Patterns will be added per the specs in [`docs/patterns/`](../../../docs/patterns/).

## Per-pattern layout

Each pattern lives under `src/<name>/` with:

```
src/<name>/
  index.ts        # public API
  types.ts        # exported types
  README.md       # usage + upgrade/drain notes
  *.test.ts       # unit tests
  fixtures/       # recorded histories for replay tests
```

## Replay-safety

Each pattern's README declares whether it is **safe to upgrade mid-flight** or **requires draining**.
