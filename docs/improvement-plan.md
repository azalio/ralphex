# ralphex Improvement Plan

Active recommendations grounded in the current repository state.

## Validation Commands

- `go test ./...`
- `go test -race -timeout=100s ./...`
- `python3 scripts/ralphex-dk.sh --test`

## Hierarchical trace trees for failed autonomous runs

**Source**: [[codetracer-towards-traceable-agent-states]] (paper note)
**Implementation Layer**: `pkg/processor`, `pkg/executor`, `pkg/progress`, `pkg/status`, and `pkg/web`
**Missing Capability**: Structured command/session trace reconstruction and failure-onset localization across task, review, external review, and finalize phases.
**Architecture Evidence**: `docs/architecture.md` defines progress logs, executor abstraction, phase/status output, dashboard/SSE sessions, Git commits, and multiple execution modes as the core runtime record.
**Benefit Hypothesis**: When a ralphex run fails after several task/review iterations, the dashboard and progress artifacts should identify the earliest failure-critical phase/command rather than only the final failing validation. Pass criteria: a synthetic failing run fixture reconstructs a hierarchical trace tree with task/review/executor nodes and marks the first command or validation step that caused the downstream failure.
**Confidence**: 0.72
**Reasoning**: ralphex already owns the orchestration shell around autonomous coding agents: it sees prompts, executor outputs, validation commands, commits, progress logs, and status phases. CodeTracer’s idea is therefore project-owned rather than generic; it turns existing progress output into a structured diagnosis surface for interrupted or failed automation.
**Why Not Already Tried**: Current architecture documents progress logs, phase status, and dashboard sessions, but it does not define a normalized trace schema, command-level event tree, or failure-onset heuristic. Existing progress is human-readable first, not a structured replay/debug artifact.

### Proposed Changes

- Define a `RunTrace` schema with nodes for plan selection, task iteration, executor call, validation command, commit, review loop, external review, finalize, and notification.
- Emit trace events from the same sites that currently write progress/status output, preserving the existing human log while adding machine-readable JSONL.
- Add a failure-onset heuristic that walks the trace backwards from the terminal error and marks the earliest failed command, unchanged review round, invalid plan parse, or validation failure that caused the run to stop.
- Surface the trace tree and failure-onset node through the web dashboard API without requiring users to parse raw progress logs.
- Add unit tests around trace event ordering and a fixture test for a synthetic validation failure followed by a review-loop abort.
