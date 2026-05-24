# Progress

Live status of work in temporal-kit. Update on every state transition (spec drafted, impl started, primitive released). Pattern-level status is also tracked in [`patterns/CATALOG.md`](patterns/CATALOG.md); this file is for *what's in motion right now*.

## Active

- **saga** — `spec` — initial draft at [`patterns/saga.md`](patterns/saga.md). 11 open questions. Next action: spec review pass to resolve or explicitly defer each open question before promoting to `impl`.

## Recently completed

_None yet._

## Up next

Ordered. Rationale per `CATALOG.md`'s "Suggested first three" — these three together exercise the conventions the rest of the catalog will follow.

1. **approval-gate** — exercises signals, timers, and typed channels. Highest-leverage primitive after saga because it shakes out signal-handling conventions end-to-end.
2. **poll-until** — exercises activity invocation defaults, cancellation, deadline tracking. The most commonly reached-for primitive in real workflows; conventions discovered here will shape every later primitive.
3. _(open)_ — choose after the first three land. See `CATALOG.md` for candidates; likely `fsm` or `exactly-once-call` based on dependency surface.

## How to update this file

- Move a primitive into **Active** when its branch is created and work is in motion.
- Move it to **Recently completed** when `CATALOG.md` flips it to `released`. Include the version and date (`YYYY-MM-DD`).
- Trim **Recently completed** to the last ~5 entries — older history lives in `CHANGELOG.md`.
- Reorder **Up next** when priorities shift; keep rationale current.
