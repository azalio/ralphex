# ralphex Improvement Plan

Active recommendations grounded in the current repository state.

## Validation Commands

- `go test ./...`
- `go test -race -timeout=100s ./...`
- `python3 scripts/ralphex-dk.sh --test`

## Clean-restart retry policy for failed executor sessions

**Source**: [[why-retrying-fails-context-contamination-in-llm-agent-pipelines]] (paper note)
**Implementation Layer**: `pkg/processor`, `pkg/executor`, `pkg/progress`, `pkg/status`, and prompt/context construction for task and review phases
**Missing Capability**: A first-class retry-attempt policy that separates clean retries from contaminated follow-up context after failed task, review, external-review, timeout, and rate-limit sessions.
**Architecture Evidence**: `docs/architecture.md` defines "Fresh execution context" as a quality goal, assigns retry limits and phase transitions to `pkg/processor`, wraps agent calls behind `pkg/executor`, and records execution state through progress logs/status files. README also says each task runs in a fresh session, while review loops can carry `{{PREVIOUS_REVIEW_CONTEXT}}` and rate-limit/session-timeout retry paths are configurable.
**Benefit Hypothesis**: When retries after executor failure rebuild a clean prompt from plan/git/progress state and quarantine failed-session output unless a policy explicitly marks it safe, repeated attempts should stop reinforcing the same bad context. Pass criteria: fixture runs that fail once via timeout, rate limit, malformed signal, and review stalemate retry with a fresh attempt id, no untrusted partial transcript in the next prompt, and progress/status output showing why context was carried or discarded.
**Confidence**: 0.67
**Reasoning**: The source paper's clean-restart result maps directly to ralphex's core product claim: long autonomous runs stay reliable because each agent session starts fresh. The current architecture already owns the orchestration and progress surfaces needed to enforce that claim, but retries are spread across task execution, Claude review, external review, plan creation, rate-limit waits, session timeouts, and manual breaks. Some paths intentionally preserve prior review context, while timeout handling already treats partial output as untrusted. Turning that implicit behavior into an explicit retry policy gives the project a measurable invariant rather than a best-effort prompt convention.
**Why Not Already Tried**: Existing plan items add trace trees, acceptance coverage, and error taxonomy, but none define a retry-attempt context contract. The code and docs mention `TaskRetryCount`, `--wait`, `--session-timeout`, `--idle-timeout`, review patience, and `{{PREVIOUS_REVIEW_CONTEXT}}`; they do not describe a normalized attempt record, clean/contaminated context classification, or tests proving failed-session output cannot leak into the next retry.

### Proposed Changes

- Define a `RetryAttempt` record emitted for every executor call with phase, attempt number, trigger (`rate_limit`, `session_timeout`, `idle_timeout`, `failed_signal`, `review_stalemate`, `manual_break`, `executor_error`), prompt fingerprint, carried-context policy, and terminal status.
- Centralize retry context construction in `pkg/processor`: clean retries rebuild from plan file, git diff/status, committed progress, and explicit validation results; failed-session stdout/stderr is excluded by default and can only be carried as summarized diagnostic text when the trigger is marked safe.
- Make the review-loop `{{PREVIOUS_REVIEW_CONTEXT}}` path policy-aware: carry resolved findings and accepted reviewer deltas, but quarantine partial timeout output and repeated unchanged findings that review-patience already classifies as stalemate.
- Surface retry hygiene in progress JSON/status/dashboard APIs so users can see "attempt 2 restarted clean after timeout" versus "attempt 2 reused prior accepted review context"; pair this with the hierarchical trace-tree plan item.
- Add table-driven tests for task retry, rate-limit retry, session timeout, idle timeout, external review retry, and manual break/resume, asserting the next prompt has a fresh attempt id and contains only policy-approved context.

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

**Source**: [[2605.211]], [[2605.213]], [[2605.214]], [[2605.215]] (ideas from vault), [[procbench-evaluating-process-level-defects-and-control-preservation-in-llm-coding-agents]] (paper note)
**Implementation Layer**: `pkg/processor`, `pkg/executor`, `pkg/progress`, `pkg/status`, `pkg/web`, and plan validation handling
**Missing Capability**: A procedure-specific error taxonomy, standardized trajectory representation, control-preservation scorecard, and latency/correctness metrics for ordered task, validation, commit, review, external-review, and finalize phases.
**Architecture Evidence**: `docs/architecture.md` defines full plan execution as an ordered flow from plan selection through task executor, validation commands, commits, review loops, and optional finalize behavior; `Core Structure` assigns phase transitions to `pkg/processor`, executor calls to `pkg/executor`, and phase/status output to `pkg/progress`, `pkg/status`, and `pkg/web`.
**Benefit Hypothesis**: Classifying each autonomous run failure by ordered-execution error type and reporting control-preservation dimensions will make stalled runs easier to debug, safer to interrupt/resume, and less likely to repeat phase-loop failures. Pass criteria: fixture runs can distinguish wrong phase order, skipped validation, repeated review stalemate, executor timeout, commit failure, and plan-parse mismatch, while dashboard/status output reports per-phase latency, terminal taxonomy code, and whether the run remained interpretable, interruptible, correctable, reversible, and handoff-ready.
**Confidence**: 0.69
**Reasoning**: The telecom procedure paper is not directly about coding agents, but its core mechanism is a strict sequence of dependent tool calls with reliability degradation as sequence length grows. ProcBench adds the directly relevant coding-agent frame: compare agents by process-level defects and control preservation, not only final success. That maps closely to ralphex's local orchestration layer because it owns the ordered run procedure, invokes external executors, runs validation, commits progress, and then enters review loops. The repo already plans trace trees and acceptance coverage, but it does not yet define stable failure classes, normalize executor trajectories into comparable events, or measure whether a failed run preserved user control.
**Why Not Already Tried**: Current plan items reconstruct traces and connect acceptance criteria to evidence. They do not classify procedure failures by sequence violation type, quantify latency as executor reasoning time plus validation/review/tool execution time per phase, or expose a compact process scorecard that says whether a human can understand, stop, correct, reverse, and take over the run.

### Proposed Changes

- Define stable `ExecutionErrorKind` values such as `plan_parse_mismatch`, `phase_order_violation`, `executor_timeout`, `validation_skipped`, `validation_failed`, `commit_failed`, `review_stalemate`, `external_review_failed`, and `finalize_failed`.
- Define a normalized `TrajectoryEvent` surface for plan selection, executor call, tool/stdout observation, validation command, commit, review verdict, retry, signal/pause, and finalize steps so ralphex can compare runs even when the underlying executor is Claude, Codex, or a custom command.
- Record per-phase start/end timestamps and executor/validation command counts in the same progress/status surfaces used by the dashboard.
- Add a `ControlPreservation` scorecard to progress JSON/status/dashboard output with five booleans or graded fields: `interpretable`, `interruptible`, `correctable`, `reversible`, and `handoff_ready`, each backed by concrete evidence such as status file freshness, clean git diff availability, last safe commit, pending phase, and recovery hint.
- Add an ordered-procedure checker in `pkg/processor` tests that asserts the expected phase sequence for full, tasks-only, review-only, external-only, and finalize-enabled runs.
- Surface the final taxonomy code and phase-latency breakdown in progress JSONL and `/api/sessions` so users can see whether a run failed because the plan was wrong, the executor stalled, validation failed, or the review loop stopped making progress.
- Add stress fixtures with increasing task/review step counts to detect reliability degradation limits before raising default iteration or review-patience settings; assert the process scorecard stays stable across recoverable failures and degrades explicitly when a run loses reversibility or handoff readiness.

## Phase precondition registry for state-constrained execution [2605.216]

**Source**: [[SDOF: Taming the Alignment Tax in Multi-Agent Orchestration with State-Constrained Dispatch]] (paper note)
**Implementation Layer**: `pkg/processor`, `pkg/plan`, `pkg/git`, `pkg/progress`, `pkg/status`, `pkg/web`, and validation-command handling
**Missing Capability**: A first-class phase legality and precondition registry that proves each ordered phase is allowed before ralphex enters it, instead of only classifying failures after an illegal or under-prepared phase already ran.
**Architecture Evidence**: `docs/architecture.md` defines full plan execution as a strict ordered flow: plan selection, git state validation, task executor, validation commands, commits, review loops, optional finalize, and completed-plan movement. It also assigns phase transitions to `pkg/processor`, git/worktree checks to `pkg/git`, progress/status output to `pkg/progress` and `pkg/status`, and dashboard visibility to `pkg/web`.
**Benefit Hypothesis**: If every task, validation, commit, review, external-review, finalize, and completed-plan move passes explicit preconditions before execution, ralphex will block invalid autonomous transitions earlier and produce clearer recovery guidance. Pass criteria: fixture runs prove skipped validation, dirty git state, missing plan task, unresolved acceptance coverage, stale worktree, review-patience stalemate, and finalize-before-clean-review are blocked with stable precondition codes and audit events before the next phase starts.
**Confidence**: 0.74
**Reasoning**: The existing taxonomy item will make failures easier to name, but state-constrained dispatch prevents a class of failures from happening in the first place. SDOF's explicit FSM plus skill-level preconditions maps cleanly to ralphex because the product is already a local workflow controller with visible ordered phases and external executor calls. This is project-owned and testable without changing the LLM executor contract.
**Why Not Already Tried**: Current plan items cover retry hygiene, trace trees, acceptance coverage, and ordered execution error taxonomy. They do not define a reusable phase-precondition registry, a pre-execution verdict schema, or dashboard/status output that says "phase blocked because the preconditions for commit/review/finalize were not satisfied."

### Proposed Changes

- Define `PhasePrecondition` and `PhasePreconditionVerdict` types with phase, check id, required state, observed evidence, blocking/severity status, and recovery hint.
- Add a phase registry for `select_plan`, `prepare_git`, `execute_task`, `run_validation`, `commit_task`, `review_first`, `review_external`, `review_second`, `finalize`, and `move_completed_plan`.
- Run precondition checks at each `pkg/processor` transition and block the next phase when required state is missing, while preserving explicit override paths for documented interactive modes.
- Emit verdicts into progress JSONL, status APIs, and dashboard session payloads so users can distinguish "phase failed after starting" from "phase was blocked before unsafe execution."
- Extend tests for full, tasks-only, review-only, external-only, worktree, and finalize-enabled flows to assert both valid phase order and blocked illegal transitions.
