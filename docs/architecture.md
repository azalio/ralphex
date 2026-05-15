# ralphex Architecture

## Overview

`ralphex` is a standalone Go CLI for autonomous plan execution in a Git repository. It reads Markdown implementation plans, creates or selects branches/worktrees, runs Claude Code or compatible executors task-by-task, performs multi-phase review loops, commits progress, and optionally serves a real-time web dashboard.

The repository is a Go module (`github.com/umputun/ralphex`) with a single primary binary in `cmd/ralphex`. It also includes shell wrappers, a static documentation site, Docker support, and CI/release workflows.

## Scope

**In scope:**
- CLI plan execution from `docs/plans/*.md` or an explicitly supplied plan file.
- Interactive plan creation through `--plan`.
- Task-only, review-only, external-only/codex-only, full, and worktree modes.
- Configurable Claude, Codex, and custom external review executors.
- Git branch/worktree/progress management and automatic commits.
- Multi-phase review prompts and custom review agents.
- Optional web dashboard and SSE progress streaming.
- Notifications through Telegram, email, Slack, webhook, or custom script.

**Out of scope:**
- Hosted orchestration service; the project runs locally or in its Docker wrapper.
- IDE plugin behavior; the README positions it as terminal-first.
- Direct model hosting; it shells out to configured agent/review tools.

## Quality Goals

- **Autonomy with auditability:** every task/review phase should leave progress logs and commits.
- **Fresh execution context:** each task runs in a new agent session to reduce long-context degradation.
- **Repository safety:** Git state, validation commands, progress files, and optional worktrees bound execution to a project checkout.
- **Configurable review rigor:** default multi-agent review can be adjusted with custom agents/prompts and external tools.
- **Recoverability:** pause/break handling, progress files, session status, and dashboard state help users understand or resume interrupted runs.

## System Context

`ralphex` runs in a developer terminal inside a Git repository. It depends on:
- Git or another configured VCS command for branch, diff, commit, and worktree operations.
- Claude Code or a configured Claude-compatible command for task and review phases.
- Codex or a custom external review script when external review is enabled.
- Local config under `~/.config/ralphex/` plus optional project-local `.ralphex/`.
- Optional notification providers and browser access to the local dashboard.

CI uses GitHub Actions to run Go tests with race detection and coverage, run the Docker wrapper test, run `golangci-lint`, and publish coverage.

## Core Structure

| Path | Responsibility |
|------|----------------|
| `cmd/ralphex/main.go` | CLI flags, startup flow, mode selection, signal handling, config loading, worktree/dashboard setup. |
| `pkg/config` | Embedded defaults, config loading/merging, prompt/agent loading, model/review settings, notification/color defaults. |
| `pkg/plan` | Plan file selection, parsing, checkbox/task manipulation, recent plan discovery, branch name derivation. |
| `pkg/processor` | Main task/review orchestration loop, executor wiring, retry limits, review patience, phase transitions. |
| `pkg/executor` | Claude, Codex, and custom command execution, output handling, process-group behavior, line reading. |
| `pkg/git` | Git/VCS operations used for branch creation, diffs, commits, and worktree behavior. |
| `pkg/progress` | Progress log writing, locking, colors, and phase/status output. |
| `pkg/status` | Runtime phase/status model shared by CLI and web surfaces. |
| `pkg/web` | Embedded dashboard, session manager, SSE, plan/status APIs. |
| `pkg/notify` | Notification integrations and custom notification hooks. |
| `scripts/*-as-claude` | Compatibility wrappers for Codex, Copilot, Gemini, OpenCode, and related executor modes. |

## Runtime Flows

**Full plan execution:**
1. Parse CLI flags and load merged global/local config.
2. Select or create a plan file.
3. Validate Git state, determine default/base branch, and optionally create an isolated worktree.
4. Parse the next incomplete `### Task N` section and checkbox steps.
5. Run the task executor with configured model/effort, timeouts, and retry behavior.
6. Run validation commands from the plan.
7. Mark completed task checkboxes, commit the task result, and repeat.
8. Run first review, external review, and second review loops as configured.
9. Optionally run finalize behavior and move the completed plan into `docs/plans/completed/`.

**Review-only/external-only flow:**
1. Compare current branch against default/base ref.
2. Run configured review phases without task execution.
3. Ask the executor to address confirmed findings and commit fixes.
4. Stop on clean review, iteration limit, or review-patience stalemate.

**Dashboard flow:**
1. `--serve` starts an HTTP server bound to `--host`/`--port`.
2. Progress logs and session state are broadcast over SSE.
3. `/api/plan` and `/api/sessions` expose parsed plan/session state for the embedded dashboard.

## Source of Truth

- **Plan state:** Markdown plan files and checkbox state in the selected plans directory.
- **Execution state:** Git commits, branch/worktree state, progress logs, and status phase files.
- **Configuration:** Embedded defaults merged with `~/.config/ralphex/` and optional `.ralphex/`.
- **Prompts/agents:** Embedded prompt and agent defaults, overridable by config directories.
- **Tests/CI:** Go unit/e2e tests plus GitHub Actions workflows define regression expectations.

## Cross-cutting Concepts

- **Mode model:** `full`, `review`, `codex-only`/external-only, `tasks-only`, and `plan` modes share executor/config plumbing but differ in phase selection.
- **Executor abstraction:** task, review, external review, and custom review commands are wrapped behind common run-result handling.
- **Model effort parsing:** task and review models accept `model[:effort]`, with review model falling back to task model.
- **Worktree isolation:** `--worktree` moves task execution into `.ralphex/worktrees/<branch>` so parallel plans can run without sharing a checkout.
- **Signal handling:** SIGINT/SIGTERM cancel execution; Ctrl+\ can pause task execution or break review loops on supported platforms.
- **Progress locking:** progress files and lock helpers coordinate dashboard/watch behavior and avoid cross-session corruption.

## Deployment/Operations

- Build/test locally with Go 1.26 and `go test ./...`.
- CI runs `go test -race -timeout=100s -covermode=atomic -coverprofile=... ./...`, wrapper tests, `golangci-lint`, and coverage upload.
- Docker release workflows build and publish images after successful CI.
- Runtime config can be initialized/reset/dumped through CLI flags such as `--init`, `--reset`, and `--dump-defaults`.
- The web dashboard defaults to `127.0.0.1:8080` and can be changed with `--host` and `--port`.

## Known Risks/Gaps

| Risk/Gap | Evidence | Impact |
|----------|----------|--------|
| External executor dependence | README and `pkg/executor` show shelling out to Claude/Codex/custom tools. | Tool login, rate limits, or CLI behavior changes can block runs. |
| Autonomous commits require clean intent | README promises automatic commits after tasks/review fixes. | Bad plans or weak validation can commit incorrect work unless review catches it. |
| Long-running review stalemates | README documents `--review-patience`. | External and primary review tools can disagree without a hard stop. |
| Worktree lifecycle complexity | README documents leftover worktrees after interruption. | Interrupted runs may require manual cleanup or review reruns inside the worktree. |
| Web dashboard is local state over embedded UI | `pkg/web` exposes local plan/session state. | It is useful for monitoring but not an authenticated multi-user service. |

## ADR Links

No dedicated ADR files were found. Architectural decisions are documented in `README.md`, package boundaries, tests, and release/CI workflows.

## Freshness

**Last refreshed:** 2026-05-15

**Refresh reason:** Daily maintenance refresh. Since the previous architecture generation, committed repository changes are synthesis-plan updates only; no committed runtime source change altered the processor, executor, progress, status, web, or notification boundaries.

**Evidence Files Used:**
- `README.md`
- `go.mod`
- `cmd/ralphex/main.go`
- `pkg/{config,plan,processor,executor,git,progress,status,web,notify}/*`
- `.github/workflows/*`
- `docs/{custom-providers,hg-support,notifications,bedrock-setup}.md`
- `scripts/*/README.md`
- `git log --since=2026-04-30`
