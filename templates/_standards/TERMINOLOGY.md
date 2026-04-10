# APM Terminology

This document defines the formal vocabulary of the APM workflow for this custom adaptation. This is a development-time specification: agents do not read this file or any `_standards/` document during runtime. The vocabulary defined here takes effect through the commands, guides, and skills that use these terms consistently. Template authors use terms exactly as defined here; agents inherit correct terminology through the templates they read.

Formal terms are always capitalized and carry defined meaning. All other language is natural - standard English capitalization applies (headings, labels, proper nouns) but confers no formal status. There is no intermediate category between formal vocabulary and natural language. Workflow definitions follow `WORKFLOW.md`.

---

## 1. Roles

| Term | Definition |
| ------ | ------------ |
| **Planner** | Gathers requirements and decomposes them into planning documents. Single instance, no Handoff. |
| **Manager** | Coordinates and orchestrates the Implementation Phase - dispatches subagents, reviews results, maintains planning documents and memory. Single role, multiple instances via Handoff. |
| **Worker Group** | A named subagent execution domain defined in the Plan (e.g., "Frontend Workers", "Backend Workers"). Represents one or more ephemeral subagents that share a domain context. Tasks are assigned to Worker Groups; the Manager spawns subagents within a group at dispatch time. |

---

## 2. Phases

| Term | Definition |
| ------ | ------------ |
| **Planning Phase** | The Planner transforms User requirements into planning documents through Context Gathering and Work Breakdown. |
| **Implementation Phase** | The Manager transforms the Spec, Plan and Rules into completed deliverables through autonomous subagent dispatch and coordination. |

---

## 3. Planning Documents

Three documents form a waterfall: Spec (what to build) → Plan (how work is organized) → Rules (how work is performed).

| Term | Definition | Location |
| ------ | ------------ | ---------- |
| **Spec** | Project-specific design decisions and constraints that inform the Plan. The Manager may update it during the Implementation Phase. | `.apm/spec.md` |
| **Plan** | Stage and Task breakdown with Worker Group assignments, Dependency Graph, and validation criteria. The Manager may update it during the Implementation Phase. | `.apm/plan.md` |
| **Rules** | Execution rules applicable to all or most Tasks, maintained as the APM Rules block within `CLAUDE.md`. Auto-loaded into all agent contexts by Claude Code. The Manager may update it during the Implementation Phase. | `CLAUDE.md` at workspace root |
| **Dependency Graph** | Mermaid diagram in the Plan header that visualizes Task dependencies, Worker Group assignments, and execution flow. Enables the Manager to identify batch candidates, parallel dispatch opportunities, and critical path bottlenecks. | Within `.apm/plan.md` |

---

## 4. Work Units

| Term | Definition |
| ------ | ------------ |
| **Stage** | Milestone grouping of related Tasks representing a coherent project progression. |
| **Task** | Discrete work unit with objective, deliverables, validation criteria, and dependencies. Tasks contain ordered sub-units (steps) that support failure tracing but have no independent validation. |
| **Dispatch Unit** | The atomic unit of subagent dispatch. A dispatch unit is either a single Task or a batch of sequential same-group Tasks, executed by one subagent. |

### Task Lifecycle States

Tasks in the Tracker progress through these states:

| Term | Definition |
| ------ | ------------ |
| **Waiting** | Dependencies not met. |
| **Ready** | All dependencies complete; can be dispatched. |
| **Active** | Dispatched to a subagent; execution in progress. |
| **Done** | Coordination decision finalized - terminal state. |

Outcome statuses are inputs to the Manager's coordination decision - the Manager reviews the outcome, investigates if needed, and then decides the lifecycle transition. A Task becomes Done when the Manager makes a terminal coordination decision - proceeding after Success, accepting a non-Success outcome, or restructuring work. A Task remains Active during investigation, while a follow-up is pending, or while the Manager is deciding how to proceed. Done is terminal; if completed work needs revisiting due to later findings, the Manager creates a new Task through plan modification rather than reopening the original. The original remains Done as a historical coordination decision; the new Task references it and captures what specifically needs correction. When all Tasks in a Stage are Done with no pending merges, the Stage collapses to complete.

### Task Outcome Statuses

Task Logs record the execution result:

| Term | Definition |
| ------ | ------------ |
| **Success** | Objective achieved, all validation passed. |
| **Partial** | Some progress made; subagent needs guidance or cannot continue. |
| **Failed** | Objective not achieved; subagent attempted but could not resolve the issue. |

Partial means "I need guidance to continue" or "User judgment is required." Failed means "I could not achieve the objective."

---

## 5. Procedures

| Term | Definition |
| ------ | ------------ |
| **Procedure** | A defined workflow operation in APM. Each Procedure covers a specific part of agent work: Context Gathering, Work Breakdown, Task Assignment, Task Execution, Task Review, Task Logging, and Handoff. |

| Term | Definition |
| ------ | ------------ |
| **Context Gathering** | Planner elicits requirements through structured question rounds and produces a consolidated summary for User review. |
| **Work Breakdown** | Planner decomposes gathered context into Spec, Plan, and Rules. |
| **Task Assignment** | Manager assesses readiness, determines dispatch mode, classifies dependencies, constructs Task Prompts, and spawns subagents via `Agent()`. |
| **Task Execution** | Subagent receives a Task Prompt within the `apm-worker` agent context (which provides behavioral rules as the system prompt), executes instructions, validates results, iterates if needed, and logs the outcome to memory. |
| **Task Review** | Manager reads subagent return value and Task Log, determines review outcome, modifies planning documents when findings warrant it, and updates the Tracker. |
| **Task Logging** | Subagent writes a structured Task Log capturing outcome, validation, deliverables, and flags, then returns a structured result summary. |
| **Handoff** | Context transfer between successive Manager instances when context window limits approach. Manager-only. |

---

## 6. Communication

The communication system consists of a single Handoff Bus file at `.apm/bus/manager/handoff.md`. During the Implementation Phase, Task Prompts are delivered via `Agent()` prompt parameters and Task Reports are received as `Agent()` return values - no file-based bus is needed for Task dispatch.

| Term | Definition |
| ------ | ------------ |
| **Handoff Bus** | Manager's bus file (`handoff.md`). Contains the handoff prompt content that instructs the incoming Manager to rebuild working context. |
| **Task Prompt** | Self-contained prompt string passed to `Agent()` providing a subagent with everything needed to execute and validate a Task. Prepended with a standard preamble. |
| **Task Report** | Structured result summary returned by the subagent as its final output. Contains status, log path, flags, and a brief summary. |

---

## 7. Memory

Memory resides in `.apm/memory/` and captures project history for progress tracking and Handoff continuity.

| Term | Definition | Location |
| ------ | ------------ | ---------- |
| **Memory** | The hierarchical file structure in `.apm/memory/` that captures project history for progress tracking and Handoff continuity. Contains the Index, Task Logs, and Handoff Logs. | `.apm/memory/` |
| **Tracker** | Live project state document containing Task tracking, version control state, and working notes. Updated by the Manager throughout the Implementation Phase as the operational view for dispatch decisions, dependency analysis, and Handoff continuity. | `.apm/tracker.md` |
| **Index** | Durable project memory containing Memory notes (persistent observations and patterns) and Stage summaries (appended after each Stage completion). | `.apm/memory/index.md` |
| **Task Log** | Structured log created by subagent after Task completion. Captures outcome, validation, deliverables, and flags. | `.apm/memory/stage-<NN>/task-<NN>-<MM>.log.md` |
| **Handoff Log** | Log created during Manager Handoff containing working context not captured elsewhere. | `.apm/memory/handoffs/manager/handoff-<NN>.log.md` |

---

## 8. Defined Concepts

These concepts are not formal capitalized terms but are clearly defined because they drive real workflow decisions.

**Task dependencies.** A Task may depend on outputs from a prior Task. Dependencies are structural relationships mapped by the Planner without context-depth classification. The Manager classifies context depth at dispatch time based on how tasks are batched:

- *Intra-batch dependency:* Producer and consumer are in the same dispatch unit (same subagent). The subagent has working familiarity from executing the producer - provide light context (recall anchors, file paths).
- *Cross-dispatch dependency:* Producer was in a different dispatch unit or completed in a previous dispatch cycle. The subagent has zero familiarity - provide comprehensive context (file reading instructions, output summaries, integration guidance).

**Dispatch modes.** The Manager determines how to dispatch Ready Tasks:

- *Single:* One Task dispatched as one subagent via `Agent(description="...", prompt=...)`.
- *Batch:* Multiple sequential Tasks dispatched as one subagent. Candidates either form a chain with only internal dependencies, or are an independent group of same-group Tasks all Ready simultaneously. When forming chains, the Manager weighs whether external Tasks depend on intermediate results - if so, dispatching individually allows earlier review and unblocks dependent work sooner. Soft guidance: 2-3 Tasks per batch.
- *Parallel:* Two or more dispatch units sent as multiple concurrent subagents when no unresolved dependencies exist among them. Multiple `Agent()` calls in a single message. Requires version control workspace isolation.

**Manager instances.** The Manager role is numbered sequentially. Manager 1 is the first Manager; Manager 2 takes over after Handoff. Instance numbers are tracked in the Tracker's working notes. Auto-compaction recovery does not increment the instance number - the recovered Manager continues as the same instance.

**Recovery.** Context reconstruction after platform auto-compaction within the Manager instance. The recovered Manager re-reads the initiation command and follows its document loading instructions to rebuild procedural knowledge and project state. Recovery does not increment the instance number or constitute a Handoff. The Manager notes the recovery in the Tracker and in its eventual Handoff Log.

**APM session.** One complete workflow cycle operating on a single set of `.apm/` artifacts. Multiple Manager instances participate across Handoffs, and the artifact set remains continuous until archival. An APM session spans at least two chat conversations (Planner and Manager).

**Session continuation.** Archiving the current session's artifacts and reinitializing for a new session. The summarization command produces an optional session summary, then the `apm archive` CLI command moves artifacts into `.apm/archives/` and removes the current installation. The user runs `apm init` (or `apm custom`) to begin a new session with fresh templates while retaining read access to archived context.

**Session archive.** A snapshot of a session's artifacts stored as a dated directory in `.apm/archives/` (`session-YYYY-MM-DD-NNN`). Contains planning documents, Tracker, Memory, and an optional session summary. The snapshot captures whatever state the session was in at archival time - completed, partial, or in-progress. The `metadata.json` file is the canonical archive marker.

**Session summary.** Optional artifact (`.apm/session-summary.md`) produced by a standalone agent via the summarization command - not a Planner or Manager. Captures a point-in-time snapshot of the session. Can be produced at any point during a session, not only after completion.

**Understanding summary.** A consolidated presentation of gathered context for User review and approval. The Planner presents one at the end of Context Gathering. The Manager presents one during first initiation. Both serve as approval gates.

---

**End of Terminology**
