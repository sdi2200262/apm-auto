# APM Workflow

This document is the formal specification of the APM workflow for this custom adaptation. It defines the phases, systems, and procedures that govern how agents coordinate to deliver project outcomes. This is a development-time specification: agents do not read this file during runtime. All commands, guides, and skills derive their behavior from this specification - the rules defined here take effect through those implementation files.

This adaptation replaces user-mediated Worker chats with Manager-driven subagent dispatch via Claude Code's `Agent()` tool. The Manager autonomously spawns ephemeral subagents to execute Tasks, reviews their output, and continues. The User participates during planning and is consulted only when genuine human judgment or external action is needed.

Terms used here are defined in `TERMINOLOGY.md`. Writing conventions follow `WRITING.md`.

---

## 1. Design

APM is a multi-agent project management framework. This adaptation targets Claude Code exclusively and replaces the platform-agnostic, user-mediated Worker model with autonomous subagent dispatch.

**Manager-autonomous coordination** - The User initiates the Planner and Manager. During the Implementation Phase, the Manager operates autonomously: spawning subagents via `Agent()`, reviewing their results, merging branches, updating the Tracker, and looping. The User is consulted only for genuine human judgment, significant plan changes, or external actions. All inter-agent communication during implementation is internal to the Manager's coordination loop.

**Claude Code native** - This adaptation targets Claude Code only. Agent dispatch uses Claude Code's `Agent(description="...", prompt=...)` tool. CLAUDE.md is auto-loaded into all agent contexts by Claude Code. The build pipeline produces a single Claude Code bundle.

**Context scoping** - Subagents are scoped by prompt content. Each subagent receives a self-contained Task Prompt with everything needed to execute - dependency context, design decisions, instructions, validation criteria. Subagents do not reference the Spec, Plan, Tracker, or Index. The Manager enforces this scoping by extracting relevant content into Task Prompts rather than referencing coordination-level documents. CLAUDE.md (Rules) is auto-loaded by Claude Code into every subagent context. The Planner holds initial project context during the Planning Phase but does not participate during implementation.

**Structured memory** - Memory captures project history in a hierarchical file structure that enables Handoff continuity and efficient progress tracking.

**Subagent usage** - Two distinct categories of subagent exist in this adaptation:

- *Task dispatch subagents:* Spawned by the Manager via `Agent(prompt=...)` to execute Tasks. These are the primary execution mechanism - ephemeral, one per dispatch unit, receiving comprehensive context in the prompt. They read the execution and logging guides, execute, write Task Logs, and return a structured result. They cannot spawn nested subagents.
- *Utility subagents:* Spawned by the Planner or Manager for focused investigation, research, or exploration via `Agent(subagent_type="Explore", prompt=...)`. These preserve the spawning agent's context window for its primary role. The Planner uses them during Context Gathering; the Manager uses them during Task Review for investigation. Findings are verified by the spawning agent before integration.

**No nested subagents** - Claude Code subagents cannot spawn other subagents. Task dispatch subagents iterate within their own context when debugging, but with a hard limit: 2-3 targeted correction attempts. If the issue persists, the subagent stops iterating, commits progress, writes a Partial Task Log with detailed diagnostics, and returns immediately. Continuing past this limit degrades output quality as context fills with failed attempts. The Manager handles escalation by spawning a fresh investigation subagent with targeted context, reframing the approach, or investigating directly.

---

## 2. Phases

### 2.1 Planning Phase

The Planner transforms User requirements into planning documents through two sequential procedures: Context Gathering, then Work Breakdown. After the User approves all three planning documents, the Planner initializes the Manager's Handoff Bus at `.apm/bus/manager/handoff.md` and directs the User to start the Implementation Phase by initiating the Manager in a new chat.

### 2.2 Implementation Phase

The Manager transforms the Plan into completed deliverables by autonomously dispatching subagents. The Implementation Phase begins when the User initiates the first Manager. The Manager determines its init path from the Handoff Bus: if the bus has content, the Manager is an incoming instance and processes the handoff; otherwise, it is the first instance. The first Manager reads planning documents and guides (including the execution and logging guides to understand subagent behavior), explores the workspace's git state, then presents an understanding summary alongside proposed version control conventions for User approval - including any Stage boundaries where holistic verification may be warranted. On approval, the Manager writes VC conventions to the Tracker and Rules, initializes the Tracker with Task tracking, and enters an autonomous coordination loop: dispatching Tasks via `Agent()`, reviewing results (return value + Task Log), merging branches, updating the Tracker, and looping. Each Task progresses through: Task Assignment → Task Execution → Task Logging → Task Review. Stages execute sequentially; Tasks within a Stage can be dispatched individually, in batches, or in parallel when version control provides workspace isolation. After all Stages complete, the Manager presents a project completion summary. When Manager context window limits approach, Handoff transfers context to a successor instance.

---

## 3. Planning Documents

| Document | Purpose | Location | Access |
| -------- | ------- | -------- | ------ |
| Spec | Define what is being built | `.apm/spec.md` | Manager reads directly; relevant content extracted into Task Prompts by Manager |
| Plan | Define how work is organized | `.apm/plan.md` | Manager reads directly; Task definitions extracted into Task Prompts by Manager |
| Rules | Define how work is performed | `CLAUDE.md` at workspace root | All agents (auto-loaded by Claude Code) |

### 3.1 Spec

The Spec defines what is being built - project-specific design decisions and constraints. It resides at `.apm/spec.md` with a free-form markdown structure determined by project needs. The Spec informs the Plan.

Subagents are not given the Spec - the Manager extracts relevant content into Task Prompts during Task Assignment, making them self-contained. The Manager may update the Spec during the Implementation Phase when execution findings warrant it.

### 3.2 Plan

The Plan defines how work is organized - Stages, Tasks, Worker Group assignments, a dependency graph, and validation criteria. It resides at `.apm/plan.md`.

The Plan contains a header (project name, Worker Groups, Stages, dependency graph) followed by Stage sections containing Tasks. Each Task specifies an objective, output, validation criteria, guidance, dependencies, and steps. The dependency graph is a mermaid diagram that visualizes Task dependencies and Worker Group assignments, enabling the Manager to identify batch candidates, parallel dispatch opportunities, and critical path bottlenecks.

Worker Groups define subagent execution domains (e.g., "Frontend Workers", "Backend Workers"). Each group represents one or more ephemeral subagents that share a domain context. Tasks are assigned to Worker Groups; the Manager determines at dispatch time whether to batch same-group tasks into a single subagent or dispatch them separately.

Task dependencies are listed in each Task's dependencies field as structural relationships. The Planner maps all natural dependencies without classifying them - dependency context depth (intra-batch vs cross-dispatch) is determined by the Manager at dispatch time based on how tasks are batched.

Subagents are not given the Plan - the Manager extracts Task content into Task Prompts during Task Assignment. The Manager may update the Plan during the Implementation Phase, updating the dependency graph when modifications affect Task dependencies.

### 3.3 Rules

Rules define how work is performed - universal execution patterns applied during Task Execution. They are stored within the APM Rules block in `CLAUDE.md` at the workspace root. Content outside this block is user-managed and preserved. When relevant standards already exist outside the block, the APM Rules block references them rather than duplicating.

All agents have direct access to this file - Claude Code auto-loads CLAUDE.md into every context, including subagent contexts. Rules are read by every subagent in the project. Content is applicable to all or most Tasks. Rules are framed conditionally on the work being performed rather than on a specific domain, so any subagent can determine whether a rule applies to its current work. Narrowly domain-specific execution patterns belong in Task Prompts via Spec extraction, not here.

Version control is the Manager's operational domain. The Manager creates a feature branch for every dispatch, subagents always operate on their assigned branch, and parallel dispatch uses worktrees for workspace isolation. The Manager establishes all VC conventions during first init: it explores the workspace's git state, reads any VC observations the Planner recorded in the Spec, and confirms conventions with the User. When no conventions are detected or specified, the Manager proposes lightweight defaults: `type: description` commits (feat, fix, refactor, docs, test, chore) and `type/short-description` branches. APM does not push to remotes by default. When `.apm/` resides inside a repository directory, the Manager adds `.apm/` to `.gitignore` by default and asks the User if they want to track any `.apm/` artifacts in git (planning documents, Memory). If the User declines version control entirely, the Manager notes that parallel dispatch is unavailable. Commit conventions are universal execution patterns (every subagent commits following the same format) and belong in Rules. Branch naming conventions are recorded in the Tracker's Version Control section. When the workspace contains multiple repositories with different conventions, per-repository conventions are captured in the Tracker and only conventions that are genuinely universal across all repositories belong in Rules.

### 3.4 Document Modification

The Spec and Plan have bidirectional influence - changes to one may require adjustments in the other. Rules are generally isolated from architecture and coordination-level documents. When modifying any planning document, the Manager assesses cascade implications before executing changes.

**Manager authority** (small contained changes):

- Single Task clarification or correction
- Adding a missing dependency
- Isolated Spec addition
- Minor Rules adjustment

**User collaboration required** (significant changes):

- Multiple Tasks affected
- Design direction change
- Scope change
- New Stage addition or major restructure
- Multiple small modifications that together represent significant change

---

## 4. Communication System

**Runtime:** `skills/apm-communication/SKILL.md`

### 4.1 Communication Models

Agent communication follows two models based on audience:

**Agent-to-user communication.** Agents explain decisions and actions to Users in natural language. No framework vocabulary - section references (§N.M), procedure step names, checkpoint labels, decision categories - is exposed. Only terms defined in `TERMINOLOGY.md` are used formally. When describing decisions, agents explain what happened, what was decided, and what happens next - not which procedure branch was taken. When directing Users to take action, agents provide specific actionable guidance naturally: which command to run, in which context, with what arguments. Communication at workflow transitions orients the User on what was completed and what comes next.

**Visible reasoning.** Agents present analysis visibly in chat so the User can understand, review, and audit their decisions - explaining assessments, justifying choices, and surfacing trade-offs. Internal reasoning may reach conclusions before visible output begins, but visible analysis in chat must still walk through the reasoning that led to those conclusions. The communication skill defines the standards.

**Agent-to-system communication.** Structured per schemas and format specifications. Artifact writing, memory logs. Governed by the communication skill and each guide's structural specifications.

### 4.2 Manager Handoff Bus

The Handoff Bus is a file-based communication mechanism at `.apm/bus/manager/handoff.md`. It is the only bus file in this adaptation. The Planner creates it at the end of the Planning Phase. It is either empty (no handoff pending) or contains a handoff prompt for the incoming Manager instance. The outgoing Manager writes the handoff prompt; the incoming Manager reads and clears it during initialization.

---

## 5. Memory

### 5.1 Structure

Project persistence resides in `.apm/` with this hierarchy:

```text
.apm/
├── tracker.md
├── bus/
│   └── manager/
��       └── handoff.md
├── memory/
│   ├── index.md
│   ├── stage-<NN>/
│   │   └── task-<NN>-<MM>.log.md
│   └── handoffs/
│       └── manager/
│           └── handoff-<NN>.log.md
```

**Tracker** (`tracker.md`) is the live project state document. It contains Task tracking, version control state, and working notes. The Manager updates it throughout the Implementation Phase as the operational view for dispatch decisions, dependency analysis, and Handoff continuity.

**Index** (`memory/index.md`) is the project's durable memory. It contains Memory notes (observations with lasting impact on future coordination) and Stage summaries (historical record of each Stage, appended after completion). Memory notes come first - forward-looking knowledge is hit first by incoming Managers.

**Task tracking** (within Tracker) records Task statuses, domain assignments, and branch state per Stage. The Manager updates it after each Task Review. Tasks progress through lifecycle states per `TERMINOLOGY.md` §4: Waiting, Ready, Active, Done.

**Working notes** (within Tracker) are coordination context accumulated during the Stage - User preferences, coordination insights, technical observations, patterns noticed during reviews. At Stage end, working notes are distilled: observations with lasting impact become Memory notes, Stage-specific observations become Stage summary prose. Working notes are inserted and removed as context evolves.

**Stage directories** (`memory/stage-<NN>/`) contain Task Logs for each Stage. Subagents write Task Logs directly to the specified `log_path`; the file write operation creates parent directories as needed.

**Task Logs** are structured logs written by subagents after Task completion. They capture outcome status, validation results, deliverables, and flags.

**Handoff Logs** (`memory/handoffs/manager/`) are logs created during Manager Handoff containing working context not captured elsewhere.

### 5.2 Task Log Flags

Subagents set flags based on what they observe during execution. The Manager interprets these flags with full project awareness:

| Flag | Interpretation |
| ---- | -------------- |
| `important_findings` | The subagent observed something potentially beyond the Task's scope. The Manager assesses whether it affects planning documents or other Tasks. |
| `compatibility_issues` | The subagent observed dependency or system conflicts. The Manager assesses whether this indicates issues with the Plan, Spec, or Rules. |

When uncertain whether a finding warrants a flag, subagents set it to true. False negatives harm coordination more than false positives.

### 5.3 Task Outcome Status

Task outcome status reflects whether the objective was achieved:

| Status | Definition |
| ------ | ---------- |
| Success | Objective achieved, all validation passed |
| Partial | Some progress made; the subagent needs guidance or cannot continue |
| Failed | Objective not achieved; the subagent attempted but could not resolve the issue |

---

## 6. Planning Phase

### 6.1 Context Gathering

**Runtime:** `commands/apm-1-initiate-planner.md` (§2), `guides/context-gathering.md`

The Planner gathers project requirements through three progressive rounds of questions, deriving technical formalization from natural User responses rather than asking Users to produce technical content directly.

**Workspace assessment** - Before question rounds begin, the Planner scans the workspace to orient itself: directory structure, git repositories (commit history, branch structure as project signals), `CLAUDE.md` contents, and the location of existing materials (documentation, requirements, specs). The Planner reads `CLAUDE.md` freely as structural orientation. When the User's initiation context references specific documents or describes the project with enough specificity to imply document authority, the Planner treats those materials as authoritative - reads them and may explore based on them. For materials discovered during scanning without initiation context authority, the Planner lists them for the User alongside the first round of questions, asking which are current and relevant. On confirmation, the Planner reads them and may use them as a basis for deeper exploration. The workspace assessment produces an initial understanding of the project environment that informs question rounds and feeds into the Spec's Workspace section during Work Breakdown.

**Round 1 - Existing Materials and Vision.** Project type, problem, scope, skills, existing documentation, current vision.

**Round 2 - Technical Requirements.** Design decisions and constraints, work structure, dependencies, technical requirements, validation criteria.

**Round 3 - Implementation Approach and Quality.** Technical constraints, workflow preferences, quality standards, coordination needs, domain organization, design decisions and constraints.

The Planner operates with progressive awareness of the workflow - understanding that gathered context feeds into three planning documents with different audiences, that the Manager handles all coordination at runtime, and that technical research needed for accurate planning must be resolved during Context Gathering rather than deferred to execution. This awareness shapes question selection toward project substance and away from coordination mechanics.

Each round follows an iteration cycle: ask initial questions, assess gaps after each response, follow up until understanding is complete, present a round summary, advance. Before advancing, the Planner includes an open-ended question targeting gaps the focused questions may have missed. When User responses reference codebase elements, the Planner proactively explores before continuing - utility subagent usage is encouraged to avoid context bloat. After subagent results return, the Planner verifies critical claims by reading referenced files or running referenced commands directly - subagent summaries compress and sometimes misrepresent details that shape planning. When verification contradicts the subagent's findings or reveals important context that needs deeper investigation, the Planner dispatches a follow-up subagent with a more targeted scope to resolve the discrepancy, then verifies the follow-up's claims in turn. The Planner then presents verified findings to the User before incorporating them into round reasoning. When gathered context reveals multiple viable approaches to an architectural or design question, the Planner presents alternatives with a recommendation rather than selecting autonomously. When exploring a codebase that is a git repository, the Planner checks git history and branch structure as project signals alongside other exploration. The Planner captures validation criteria for each requirement, proposing concrete measures when the User does not specify them. Version control signals and coordination preferences observed during Context Gathering are noted as factual observations - placement into planning documents happens during Work Breakdown. Context Gathering produces signals and constraints about the project - never decomposition structures, agent assignments, or planning vocabulary (Stages, Tasks, Workers, tracks, phases, task sizing).

After all rounds, the Planner presents a consolidated understanding summary for User review. Modifications loop back through targeted follow-ups. User approval triggers transition to Work Breakdown.

### 6.2 Work Breakdown

**Runtime:** `commands/apm-1-initiate-planner.md` (§3-4), `guides/work-breakdown.md`

The Planner decomposes gathered context into planning documents through visible reasoning - thinking is presented in chat before file output. The User sees decomposition decisions and can redirect before artifacts are written. The Planner's workflow awareness deepens at this stage: understanding how each document is consumed (the Manager extracts Spec content per-Task into Task Prompts, enriches them at runtime, and subagents focus on their Task Prompt and Rules by design), shaping content placement decisions.

**Sequence:**

1. **Spec Analysis** - The Planner analyzes design decisions, writes the Spec (including a Workspace section capturing the project environment: directory structure, repositories, reference repositories, authoritative documents, and existing Rules file content), and presents it for User approval. The Planner may include blockquote notes after the Spec header separator with observations about the project environment for the Manager: version control observations, workspace constraints, and User preferences that affect execution.
2. **Plan Analysis** - The Planner identifies work domains and maps them to Worker Groups. It reasons about how domain characteristics create stage boundaries, then walks through each Stage's deliverables and how they decompose into Tasks before deepening the analysis per Task. Dependencies are mapped as structural relationships without context-depth classification - the Manager classifies at dispatch time. After dependency verification and completeness checks, the Planner writes the full Plan and presents it for User review and approval. The Planner may include blockquote notes after the Plan header separator with work structure observations: why certain boundaries exist, natural groupings or sequencing patterns, critical path, convergence points, and Stage boundaries where holistic verification may be valuable. Notes are awareness for the Manager - observations and reasoning that inform judgment, not instructions.
3. **Rules Analysis** - The Planner extracts universal execution patterns from gathered context (excluding version control conventions, which the Manager handles), writes the APM Rules block, and presents it for User approval.

**Task decomposition principles.** Each Task produces a meaningful deliverable with clear boundaries, scoped to a single Worker Group's domain, with specified validation criteria. Tasks should be scoped for completion within a single subagent's context window - there is no handoff mechanism for subagents. Steps within Tasks support failure tracing but have no independent validation. Decomposition granularity adapts to project size and complexity - smaller projects warrant lighter breakdown.

---

## 7. Implementation Phase

### 7.1 Task Assignment

**Runtime:** `guides/task-assignment.md`

The Manager assesses readiness, determines dispatch mode, constructs Task Prompts, sets up version control, and spawns subagents via `Agent()`.

**Dispatch assessment** - The Manager identifies Ready Tasks from the Tracker, groups them by Worker Group, and forms dispatch units. Three dispatch modes exist:

| Mode | Description | Prerequisites |
| ---- | ----------- | ------------- |
| Single | One Task dispatched as one subagent | Task is Ready |
| Batch | Multiple Tasks dispatched as one subagent | Tasks form a sequential chain or are independent same-group Tasks all Ready simultaneously |
| Parallel | Dispatch units sent as multiple concurrent subagents | No unresolved dependencies among units; version control workspace isolation |

Before dispatching, the Manager checks whether a pending review would unlock Tasks that combine well with the current unit - if waiting costs little and a plausible combination exists, the Manager may wait. When no Tasks are Ready but active subagents have not yet returned, the Manager communicates what was processed, what is pending, and what comes next.

**Dependency context classification** - The Manager classifies each dependency at dispatch time based on the dispatch plan:

- *Intra-batch:* Producer task is earlier in the same batch (same subagent). The subagent has working familiarity from executing the producer. Light context: recall anchors, key file paths, brief reference to previous work.
- *Cross-dispatch:* Producer task was in a different dispatch unit or completed in a previous cycle. The subagent has zero familiarity. Comprehensive context: file reading instructions, output summaries, integration guidance.

The Manager traces dependency chains upstream when ancestors established patterns, schemas, or contracts the current Task must follow.

**Per-Task analysis** - For each Task, the Manager synthesizes content from three sources into the Task Prompt: dependency context (classified at dispatch time per above), relevant Spec content (design decisions and constraints for this Task, extracted inline), and Plan Task fields (objective, steps, guidance, output, validation criteria). Rules are not included in Task Prompts - subagents read `CLAUDE.md` directly and the Task Prompt assumes those standards are in effect. Content from planning documents and authoritative sources is embedded directly - the Manager never references the Spec or Plan by path. Content that exists in the codebase (source files, patterns, configurations) is referenced through targeted reading instructions.

**Task Prompt construction** - The Manager assembles each Task Prompt as a self-contained document, prepends the subagent preamble (which instructs the subagent to read the execution and logging guides, notes that CLAUDE.md is auto-loaded, and states the no-nested-subagent constraint), sets up version control (feature branch for sequential, worktree for parallel), and spawns via `Agent(description="...", prompt=...)`. For batch dispatch, the prompt contains a batch envelope with multiple Task Prompts. For parallel dispatch, multiple `Agent()` calls are made in a single message (Claude Code runs them concurrently).

**Follow-up Task Prompts** - When a Task Review determines retry is needed, the Manager issues a follow-up. The default approach is a fresh `Agent()` call with a refined prompt containing what was tried, what went wrong, and what to do differently. If the experimental `SendMessage` capability is available and the subagent made good progress (Partial with clear remaining work), the Manager may continue the existing subagent for efficiency. The follow-up uses the same log path as the original (the subagent overwrites the previous log).

### 7.2 Task Execution

**Runtime:** `guides/task-execution.md`, `guides/task-logging.md`

The subagent reads the execution and logging guides per the preamble, executes Task instructions, validates results, iterates if needed, logs the outcome, and returns a structured result.

**Execution flow** - The subagent integrates dependency context if present, executes steps sequentially, then validates per the Task Prompt's validation criteria. The subagent validates autonomously first (running checks, verifying outputs), then uses Partial status when criteria require User involvement (judgment or action) - the Manager mediates User interaction. When validation fails, the subagent investigates the root cause before attempting a correction - reading error output, tracing the failure, and identifying what specifically went wrong. Subagents cannot spawn nested subagents and have a hard iteration limit of 2-3 targeted correction attempts. If the issue persists after this limit, the subagent stops iterating immediately, commits progress, writes the Task Log with Partial status and detailed diagnostics (error output, what was investigated, what was tried, root cause hypothesis), and returns. The Manager handles escalation - it can spawn investigation subagents, reframe the approach, or restructure the Task.

**Batch execution** - When receiving a batch of Tasks (multiple Task Prompts in a single prompt), the subagent executes them sequentially. After each Task, a Task Log is written immediately before proceeding to the next. If any Task results in a Failed status, execution stops. After completing all Tasks (or stopping on failure), the subagent returns a batch result with per-Task outcomes.

**Completion** - After execution, the subagent commits work to the assigned branch following conventions from Rules (if version control is active), writes a Task Log, and returns a structured result summary as its final output. For large Tasks, subagents may commit at logical intermediate points during execution rather than only at completion.

### 7.3 Task Review

**Runtime:** `guides/task-review.md`

The Manager reviews subagent results, determines review outcomes, modifies planning documents when needed, and updates the Tracker.

**Result processing** - The Manager reads the subagent's return value (structured result summary) directly from the `Agent()` call. For batch results, each Task's outcome is processed individually. Unstarted Tasks from a stopped batch re-enter the dispatch pool.

**Log review** - The Manager reads the Task Log, interprets status, flags, and body content. Consistency between claimed status and actual content is assessed.

**Drift detection** - The Manager checks whether the Task Log's content matches the dispatched objective, whether committed files correspond to the expected output, and whether there are changes outside the Task's scope. Drift indicates the subagent deviated from the assignment.

**Review outcome** - If the log shows Success with no flags and the content supports the status, the Manager proceeds. If version control is active and the subagent's Task was successful but changes remain uncommitted on the task branch, the Manager commits on behalf following the conventions from Rules. If something needs attention (flags raised, non-Success status, inconsistencies, or drift), the Manager investigates - directly for contained checks, using a utility subagent for context-intensive issues. After investigation, three outcomes are possible: no issues found, follow-up needed (the Manager issues a follow-up Task Prompt via fresh `Agent()` or `SendMessage`), or planning document modification needed (the Manager assesses cascade per §3.4 Document Modification). When investigation reveals deficiencies in previously-Done work, the Manager creates a new Task through plan modification rather than reopening the original - Done is terminal per `TERMINOLOGY.md` §4. Every outcome path ends with updating the Tracker. During parallel dispatch, all `Agent()` calls return together; the Manager processes each result, merges completed branches, reassesses readiness, and dispatches newly Ready Tasks.

**Stage verification** - After all Tasks in a Stage complete, the Manager assesses whether the Stage's deliverables require holistic verification before proceeding - based on what the User confirmed during the understanding summary, observations accumulated during Task Reviews, and Planner notes. When verification is needed, the Manager executes checks and examines the deliverables. When verification reveals issues, the Manager responds based on scope: fixing contained issues directly, dispatching a utility subagent for focused investigation, creating a new Task through Plan modification for subagent-level work, or presenting findings to the User when the direction is unclear.

**Stage summary** - After all Tasks in a Stage complete and any verification concludes, the Manager reviews the Stage's Task Logs and appends a Stage summary to the Index capturing Stage-level outcomes, notable findings, and references to individual logs.

---

## 8. Handoff and Continuity

### 8.1 Handoff

**Runtime:** `commands/apm-3-handoff-manager.md`

Handoff transfers context between successive Manager instances when context window limits approach. The Planner operates as a single instance and does not perform Handoff. Subagents are ephemeral and do not perform Handoff - each dispatch is a fresh instance. Handoff is User-initiated and can occur at any point as long as the handoff prompt captures comprehensive current state.

**Two artifacts** - Handoff produces two artifacts with distinct purposes:

| Artifact | Location | Content | Lifecycle |
| -------- | -------- | ------- | --------- |
| Handoff prompt | Handoff Bus | Current state and continuation: outstanding Tasks, mid-review progress, pointers to logs, continuation guidance | Ephemeral; cleared after incoming Manager processes it |
| Handoff Log | Memory | Past actions: working context, decisions made, approaches tried, technical notes | Persistent archival context |

**Context rebuilding** - An incoming Manager reads: the Handoff Log, the Tracker, the Index (Stage summaries and Memory notes), planning documents, guides, and relevant recent Task Logs to reconstruct working context.

### 8.2 Recovery

**Runtime:** `commands/apm-5-recover.md`

Recovery reconstructs context after platform auto-compaction, manual compaction, or a cleared or lost conversation. The User invokes the recovery command. The Manager re-reads its initiation command and follows its document loading instructions to rebuild procedural knowledge, then explores project artifacts and the codebase to reconstruct operational state. Recovery does not increment the instance number. The Manager notes the recovery event in subsequent communications and in its eventual Handoff Log.

### 8.3 Session Continuation

**Runtime:** `commands/apm-4-summarize-session.md`, `agents/apm-archive-explorer.md`, `guides/context-gathering.md` (§3.1)

Session continuation archives the current session's artifacts for future reference. After archival, the user reinitializes APM to begin a new session with fresh templates while retaining read access to previous session context.

**Archive structure** - Archived sessions reside in `.apm/archives/`. Each archive is a directory named `session-YYYY-MM-DD-NNN` (zero-padded daily counter) containing the session's planning documents, Tracker, and Memory. The bus directory is not archived - bus state is ephemeral and session-specific.

```text
.apm/archives/
├── index.md
├── session-2026-03-04-001/
│   ├── metadata.json
│   ├── plan.md
│   ├── spec.md
│   ├── tracker.md
│   ├── session-summary.md     # optional
│   └── memory/
│       ├── index.md
│       ├── stage-01/
│       └── handoffs/
└── session-2026-03-05-001/
    └── ...
```

**Archive marker** - `metadata.json` within each archive directory is the canonical archive marker. Its presence identifies a valid archive. It contains the original installation metadata with archival timestamp.

**Session summary** - An optional artifact (`session-summary.md`) produced by a standalone agent via the summarization command. When present, it provides a point-in-time snapshot of the session: project scope, stage outcomes, key deliverables, notable findings, known issues, and current codebase state including how deliverables relate to `.apm/` artifacts. Can be produced at any point during a session, not only after completion. When absent, the archive contains raw artifacts sufficient for future Planners to examine.

**Archive index** - `.apm/archives/index.md` is a table listing all archived sessions with date, scope, stages, and tasks. The summarization command updates it; if absent or malformed, it is recreated.

**Planner archive detection** - During Context Gathering (§6.1), the Planner checks for `.apm/archives/`. If archives exist, the Planner presents them to the User and asks about relevance. If the User indicates archives are relevant, the Planner uses the `apm-archive-explorer` custom subagent to examine indicated archives, then verifies findings against the current codebase before integrating into question rounds.

**Manager completion recommendation** - After project completion (§2.2), the Manager recommends running the summarization command in a new chat to produce a session summary. The summarization agent also offers archival as a follow-up step; archival is also available directly via the CLI (`apm archive`).

**`apm-archive-explorer` subagent** - A custom subagent shipped with APM bundles. It understands archive structure and efficiently extracts relevant context from archived sessions. The Planner spawns it during Context Gathering when archived sessions are relevant.

**No secondary archival** - Archives accumulate in `.apm/archives/`. There is no mechanism to archive archives. Users manage cleanup manually.

---

**End of Workflow Specification**
