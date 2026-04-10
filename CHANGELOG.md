# Changelog

All notable changes to this project will be documented in this file.

---

## [1.0.0] - 2026-04-10

Custom adaptation of [APM v1.0.0](https://github.com/sdi2200262/agentic-project-management) for Claude Code with autonomous subagent dispatch.

### Architecture

* **Manager-autonomous coordination** - The Manager spawns ephemeral subagents via Claude Code's `Agent()` tool to execute Tasks, reviews their results, and continues without User mediation between dispatch and review.
* **Custom `apm-worker` agent definition** - Subagent behavioral rules (execution procedure, iteration limits, logging formats) are defined in a Claude Code agent file loaded as the system prompt. The `disallowedTools: Agent` field enforces no-nesting at the platform level.
* **Foreground-default dispatch** - All subagent dispatch runs in the foreground by default. Background dispatch requires User confirmation of proper Claude Code permission configuration.
* **Simplified dependency model** - Dependencies are structural relationships mapped by the Planner without context-depth classification. The Manager classifies as intra-batch (light) or cross-dispatch (comprehensive) at dispatch time based on the dispatch plan.
* **Worker Groups** - The Plan defines Worker Groups (e.g., "Frontend Workers") as domain labels for subagent dispatch grouping, replacing persistent Worker chat instances.

### Changes from Official APM

* **Removed** - Worker commands (initiate-worker, check-tasks, check-reports, handoff-worker), Worker registration, Worker tracking, Worker Handoff, cross-agent overrides, non-APM agent participation, Task Bus, Report Bus
* **Added** - `apm-worker` agent definition, `TASK_DISPATCH_GUIDANCE` placeholder, foreground-default dispatch rule, hard iteration limit (2-3 attempts) with mandatory Partial return, drift detection in Task Review
* **Modified** - Commands renumbered 1-5, build targets reduced to Claude Code only, communication skill stripped to Manager Handoff Bus, Tracker template uses Domain column
* **Removed files** - `src/` (CLI source), `SECURITY.md`, `VERSIONING.md`, `templates/_standards/NOTES.md`, `bus-integration.md`, `task-execution.md` and `task-logging.md` (replaced by `apm-worker.md`)
