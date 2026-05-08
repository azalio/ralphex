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

## Acceptance-criteria IDs for plan execution coverage

**Source**: [[acai-sh]] (article note)
**Implementation Layer**: `pkg/plan`, `pkg/processor`, `pkg/progress`, `pkg/status`, `pkg/web`, and validation-command handling
**Missing Capability**: Stable acceptance-criteria identifiers that connect Markdown plan requirements to executor prompts, validation commands, review findings, commits, and dashboard status.
**Architecture Evidence**: `docs/architecture.md` defines Markdown plan files, task checkbox state, validation commands, automatic commits, multi-phase review loops, progress logs, and dashboard/SSE status as the runtime source of truth.
**Benefit Hypothesis**: If each executable plan task can declare durable acceptance IDs and ralphex records which IDs were implemented, validated, reviewed, and committed, autonomous plan runs will produce auditable acceptance coverage instead of only "task done" checkboxes. Pass criteria: a fixture plan with missing or failing acceptance IDs blocks completion, while a passing run shows per-ID coverage in progress JSONL and the dashboard.
**Confidence**: 0.70
**Reasoning**: ralphex owns the layer that turns plans into autonomous coding-agent execution. The architecture already emphasizes autonomy with auditability, fresh execution contexts, validation commands, progress logs, commits, and review loops, but those surfaces are task/phase oriented rather than requirement oriented. The ACAI idea is useful here because it gives the processor a stable join key between the plan, validation output, review output, and final commit evidence.
**Why Not Already Tried**: Current plan parsing is centered on `### Task N` sections and checkbox manipulation. The architecture does not define an acceptance-ID schema, coverage model, validation-command mapping, or dashboard view that proves a specific requirement was satisfied before a task is marked complete.

### Proposed Changes

- Extend the plan parser to recognize optional acceptance IDs under each task, for example `AC-001`, `AC-002`, or `TASK-1.1`, while preserving existing Markdown checkbox behavior for older plans.
- Thread acceptance IDs into executor prompts, validation-command summaries, review prompts, and progress logs so each phase can cite the requirement it is satisfying or challenging.
- Add a coverage accumulator in `pkg/processor` that records, per acceptance ID, implementation status, validation command evidence, review verdict, and commit hash.
- Surface acceptance coverage in the web dashboard and status APIs alongside task/phase state, including a clear blocked state for IDs with missing validation or unresolved review findings.
- Add fixture tests for plan parsing, validation-output attribution, and a synthetic run where one acceptance ID remains uncovered and prevents task completion.

## Ordered execution reliability taxonomy for autonomous plan phases [2605.213]

**Source**: [[2605.211]], [[2605.213]], [[2605.214]], [[2605.215]] (ideas from vault)
**Implementation Layer**: `pkg/processor`, `pkg/executor`, `pkg/progress`, `pkg/status`, `pkg/web`, and plan validation handling
**Missing Capability**: A procedure-specific error taxonomy and latency/correctness metrics for ordered task, validation, commit, review, external-review, and finalize phases.
**Architecture Evidence**: `docs/architecture.md` defines full plan execution as an ordered flow from plan selection through task executor, validation commands, commits, review loops, and optional finalize behavior; `Core Structure` assigns phase transitions to `pkg/processor`, executor calls to `pkg/executor`, and phase/status output to `pkg/progress`, `pkg/status`, and `pkg/web`.
**Benefit Hypothesis**: Classifying each autonomous run failure by ordered-execution error type will make stalled runs easier to debug and reduce repeated phase-loop failures. Pass criteria: fixture runs can distinguish wrong phase order, skipped validation, repeated review stalemate, executor timeout, commit failure, and plan-parse mismatch, while dashboard/status output reports per-phase latency and terminal taxonomy code.
**Confidence**: 0.69
**Reasoning**: The telecom procedure paper is not directly about coding agents, but its core mechanism is a strict sequence of dependent tool calls with reliability degradation as sequence length grows. That maps closely to ralphex's local orchestration layer: it owns the ordered run procedure, invokes external executors, runs validation, commits progress, and then enters review loops. The repo already plans trace trees and acceptance coverage, but it does not yet define stable failure classes or measure where long ordered procedures degrade.
**Why Not Already Tried**: Current plan items reconstruct traces and connect acceptance criteria to evidence. They do not classify procedure failures by sequence violation type, nor do they quantify latency as executor reasoning time plus validation/review/tool execution time per phase.

### Proposed Changes

- Define stable `ExecutionErrorKind` values such as `plan_parse_mismatch`, `phase_order_violation`, `executor_timeout`, `validation_skipped`, `validation_failed`, `commit_failed`, `review_stalemate`, `external_review_failed`, and `finalize_failed`.
- Record per-phase start/end timestamps and executor/validation command counts in the same progress/status surfaces used by the dashboard.
- Add an ordered-procedure checker in `pkg/processor` tests that asserts the expected phase sequence for full, tasks-only, review-only, external-only, and finalize-enabled runs.
- Surface the final taxonomy code and phase-latency breakdown in progress JSONL and `/api/sessions` so users can see whether a run failed because the plan was wrong, the executor stalled, validation failed, or the review loop stopped making progress.
- Add stress fixtures with increasing task/review step counts to detect reliability degradation limits before raising default iteration or review-patience settings.
