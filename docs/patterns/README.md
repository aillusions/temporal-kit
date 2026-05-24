# Patterns catalog

Language-agnostic specs for the primitives in temporal-kit. **This is the source of truth.** TypeScript and (future) Go implementations follow these documents.

## Why a spec lives here, not in code

- Forces the design conversation before the implementation conversation.
- Keeps the future Go port honest — same states, same guarantees, same failure modes.
- Replay-safety notes belong with the spec, not buried in code comments.

## Format

Each pattern is a single Markdown file: `docs/patterns/<name>.md`. Recommended sections:

- **Purpose** — one paragraph.
- **States** — explicit FSM, if relevant.
- **API surface** — names, signatures, semantics. Language-agnostic.
- **Guarantees** — what callers can rely on (ordering, at-most-once, etc.).
- **Failure modes** — what happens on cancel, timeout, activity failure, worker crash.
- **Replay-safety notes** — what changes to this primitive break in-flight workflows.
- **Open questions** — things deliberately left unspecified.

## Status

Empty. Add specs here before adding implementations.
