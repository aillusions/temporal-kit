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

## Terminology

Everything user-facing in this repo is a **primitive** — a self-contained, replay-safe building block a workflow author imports and uses. Every primitive gets a spec doc at `docs/patterns/<name>.md` and is listed in [`docs/patterns/CATALOG.md`](docs/patterns/CATALOG.md). Specs scale with complexity: `saga` warrants 400 lines, `deadliner` might warrant 50. Same shape, different length.

The only thing that does NOT get a spec is something genuinely internal — a private helper inside a primitive's source that isn't exported. Those live by doc-comment.

**Packages** (layout, not status):
- `packages/ts/core/` — `@signus/temporal-kit-core`. Foundational primitives that other primitives depend on (e.g., `workflow-mutex`, `deadliner`, `interruptible-scope`).
- `packages/ts/patterns/` — `@signus/temporal-kit-patterns`. Composite primitives that structure a whole workflow (e.g., `saga`, `fsm`, `approval-gate`).
- `packages/ts/testing/` — `@signus/temporal-kit-testing`. Replay test harness, fixture loaders, observability primitives.
- `packages/ts/examples/` — `@signus/temporal-kit-examples`. Example workflows. Private, not published.

The `core/` vs. `patterns/` split is about dependency depth, not importance — both are equally first-class to users. The rule is one-way: a `core/` primitive must not depend on a `patterns/` primitive.

## Working process

### Spec-first rule

Every new primitive starts as `docs/patterns/<name>.md` and is reviewed before any code lands. No exceptions. `docs/patterns/` is the source of truth and language-agnostic — it's what keeps the future Go port honest. If the TS impl drifts from the spec, the spec wins or the spec gets updated; the impl is never the canonical reference.

### Pipeline per primitive

1. **Spec draft** — `docs/patterns/<name>.md` per the template in [`docs/patterns/README.md`](docs/patterns/README.md). Lands in its own PR; no implementation in the same PR.
2. **Spec review** — at least one human reviewer signs off. Open questions in the spec are explicit; nothing decided silently.
3. **Implementation** — source under `packages/ts/patterns/src/<name>/`.
4. **Tests** — unit tests for the state machine; replay tests against committed history fixtures.
5. **Fixtures** — record at least one history per major branch in the state machine (happy path, failure/compensation, cancel, partial failure where applicable).
6. **Per-pattern README** — usage, upgrade-vs-drain note, link back to the spec.
7. **Catalog status update** — `docs/patterns/CATALOG.md` flipped to `released`.
8. **CHANGELOG entry** — under the version that ships it.

### Where state lives

- **`docs/patterns/CATALOG.md`** — single source of truth for every primitive's status (`planned` / `spec` / `impl` / `released` / `deferred`).
- **`docs/patterns/<name>.md`** — pattern-local design notes and open questions.
- **`docs/PROGRESS.md`** — active work, recently completed, what's up next.
- **`docs/adr/`** — cross-cutting decisions (repo conventions, versioning policy, cross-language rules). Pattern-local decisions stay in the pattern spec.

### Branching and PRs

- One primitive per PR. Mixing primitives in a single PR is a smell.
- Branch name: `pattern/<name>` (e.g., `pattern/saga`, `pattern/poll-until`).
- Commit prefixes signal the pipeline stage: `spec(saga): ...`, `impl(saga): ...`, `tests(saga): ...`, `docs(saga): ...`.

### Where decisions get recorded

- **Repo-wide:** `docs/adr/NNNN-<slug>.md` (numbered ADRs).
- **Pattern-local:** the pattern's spec doc — specifically its Open Questions section. Resolved questions move out of "Open Questions" into the relevant body section above.

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

## How to resume work

Read these in order before making any changes:

1. **`CLAUDE.md`** (this file) — scope, rules, process.
2. **`docs/PROGRESS.md`** — what's active, what was recently completed, what's up next.
3. **For the active pattern: `docs/patterns/<name>.md`** — the spec, including its Open Questions section.
4. **`docs/patterns/CATALOG.md`** — status of related patterns this work may depend on.
5. **`git log --oneline pattern/<name>`** (if the branch exists) — in-progress commits.
6. **Confirm the next concrete action with the human before editing.** If `PROGRESS.md` and the spec disagree, ask before assuming.

## Definition of done

A primitive transitions from `impl` to `released` only when ALL of the following hold:

- Spec doc (`docs/patterns/<name>.md`) has no unresolved open questions.
- Implementation matches the spec; any deviation is reflected in the spec, not the other way around.
- Unit tests pass.
- Replay tests pass against committed fixtures in `fixtures/`.
- Per-pattern `README.md` exists with usage and the upgrade-vs-drain declaration.
- `docs/patterns/CATALOG.md` status updated to `released`.
- `CHANGELOG.md` entry added under the matching version.
- Public API surface reviewed for replay-safety implications. Any removed or signature-changed export triggers a major bump.
