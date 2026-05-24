# temporal-kit

Workflow-side primitives for Temporal — sagas, approval gates, `pollUntil`, FSMs, and other patterns too fiddly to rewrite per workflow.

## What this repo is NOT

- Not a Temporal SDK wrapper. We sit on top of `@temporalio/workflow`, not in front of it.
- Not infrastructure (no worker bootstrap, no client setup, no deployment helpers).
- Not activity code. Activities are application-specific; this repo is workflow-side only.

If a primitive can't be expressed purely in workflow code, it doesn't belong here.

## Prime directive: replay safety

Every primitive runs inside Temporal's deterministic replay. Any behavior change — even one that looks like a pure refactor — can break in-flight workflows whose history was recorded against the old code.

Rules:
- Every PR that touches a primitive **must** include replay tests against committed history fixtures in that primitive's `fixtures/` directory.
- New behavior goes behind a new API surface or an explicit version branch — never a silent change to existing code paths.
- If you can't prove replay-compat, assume it's broken and bump the major.

## Spec-first workflow

New primitives start as a doc, not code:

1. Write `docs/patterns/<name>.md` describing: states, guarantees, API surface, replay-safety notes, failure modes.
2. Review the spec.
3. Then implement in `packages/ts/...`.

`docs/patterns/` is the source of truth and language-agnostic — it's what keeps the future Go port honest. If the TS impl drifts from the spec, the spec wins or the spec gets updated; the impl is never the canonical reference.

## Layout

```
packages/ts/{core,patterns,testing,examples}
packages/go/                  # placeholder, not yet implemented
docs/patterns/<name>.md       # spec per primitive (source of truth)
docs/adr/                     # architecture decisions
```

Per-primitive layout inside `packages/ts/patterns/<name>/`:

```
index.ts        # public API
types.ts        # exported types
README.md       # usage + upgrade/drain notes
*.test.ts       # unit tests
fixtures/       # recorded histories for replay tests
```

## Versioning

SemVer. Be aggressive about majors — silent behavior changes are worse than a version bump.

Each primitive's `README.md` must state one of:
- **Safe to upgrade mid-flight** — new code can replay old histories.
- **Requires draining** — running workflows must complete before deploying the new version.

If unsure, it requires draining.

## Testing

Two tiers, both required:
- **Unit tests**: pure logic, no Temporal runtime needed where possible.
- **Replay tests**: load fixtures from `fixtures/`, run the workflow against committed history, assert it replays clean.

A primitive without replay fixtures is not ready to merge.

## Cross-language rule (TS ↔ Go)

- **Names and semantics align.** A `pollUntil` in Go does what `pollUntil` in TS does. State names, guarantees, and failure modes match the spec in `docs/patterns/`.
- **Idioms do not.** Don't force Go to look like TS or vice versa. Channels, contexts, error returns — use them when porting. Same in reverse for TS conventions.

The spec is the contract; the implementation is idiomatic.

## Public API discipline

- No removing exported symbols without a deprecation cycle (one minor at minimum, called out in CHANGELOG and the primitive's README).
- Type-level breaking changes count as breaking changes.
- Re-exports from `index.ts` are the API; anything else is internal regardless of whether TS lets you import it.

## Dependencies

- `@temporalio/workflow` is a **peer dependency**. Never pin the SDK version for consumers — they choose.
- Avoid runtime deps in `core` and `patterns` packages unless there's a strong reason. Replay safety is easier with fewer moving parts.
- `testing` and `examples` packages can be looser.
