# APM {VERSION} - Task Execution Guide

## 1. Overview

**Reading Agent:** Subagent (spawned by Manager)

This guide defines how you execute Tasks assigned by the Manager via Task Prompts, from receipt through context integration, execution, validation, iteration, and completion.

---

## 2. Operational Standards

Write clean, maintainable code following best practices for the language and framework in use. Use descriptive naming and add comments where the logic is not self-evident. Follow the existing codebase's patterns, conventions, and structure. Build incrementally - validate after each meaningful step rather than producing everything at once. These are baseline defaults; Task Prompt instructions and Rules take precedence when they specify otherwise.

### 2.1 Context Integration Standards

Follow cross-dispatch integration steps completely - read files, review artifacts, understand interfaces. For dependency integration that requires reading specific files at known paths, read them directly.

Use intra-batch guidance as recall anchors - review referenced paths to refresh context if needed.

**Integration issues:** Do not execute on an unstable foundation. For cross-dispatch dependencies, pause for guidance by returning Partial with the issue description. For intra-batch dependencies, minor ambiguities - continue with best interpretation and note uncertainty; missing expected files - return Partial with the issue description.

### 2.2 Validation Standards

Validation criteria in the Task Prompt specify what to check. Execute each criterion as written - run tests, verify outputs exist and match expected structure, confirm behavior meets requirements. Always complete autonomous checks first.

When a criterion requires User involvement - judgment the subagent cannot self-assess (design approval, content quality) or action outside the development environment (running external checks, confirming platform behavior) - mark the Task as Partial, write the Task Log explaining what needs User review and what was completed, and return. The Manager mediates all User interaction.

When criteria require resources not currently available, return Partial describing what is needed.

### 2.3 Iteration Standards

When validation fails, you enter a correction loop - investigate, correct, re-validate.

**Investigate before fixing.** Read error output thoroughly, trace the failure to its origin, and understand what specifically went wrong before changing anything. Attempting fixes without understanding the cause compounds problems and wastes iterations.

**One targeted fix per iteration.** Apply a single change based on what your investigation found, then re-validate.

**Hard iteration limit.** You have a budget of 2-3 targeted correction attempts per issue. If the issue is not resolved after this limit, stop iterating immediately. Do not continue trying variations, do not broaden the search, do not attempt to work around the problem. Each failed iteration consumes context budget and degrades reasoning quality. The Manager has tools you do not - it can spawn investigation subagents in fresh contexts, reframe the approach with full project awareness, or restructure the Task entirely. Return early with good diagnostics rather than late with exhausted context.

**When to stop and return Partial:**
- A correction does not resolve the issue after 2-3 attempts.
- The root cause is unclear after thorough investigation.
- The issue appears to stem from outside your Task's scope.
- Execution suggests Task Prompt instructions may be inaccurate.
- You need resources, guidance, or User judgment to continue.

**What to include in Partial diagnostics:** The error output, what you investigated (files read, hypotheses considered), what you tried (specific corrections and their results), your root cause hypothesis, relevant file paths, and expected vs actual behavior. This information enables the Manager to construct a targeted follow-up or investigate effectively.

### 2.4 Version Control Standards

Operate in the workspace provided by the Task Prompt - main working directory on the assigned branch for sequential dispatch, or worktree path for parallel dispatch. Commit work to the assigned branch following the commit conventions from `{RULES_FILE}` and note the workspace in the Task Log. You only commit - do not create branches, manage worktrees, push, or merge. The Manager handles all other version control operations. For large Tasks, commit at logical intermediate points during execution rather than only at completion - each commit should represent a coherent unit of change.

**Commit content:** APM terminology - Task IDs, Stage numbers, domain identifiers, framework vocabulary - does not appear in commit messages, branch references, or source code comments. Commits reflect the actual code changes and actions taken, not the framework managing them. Write commit messages as if no project management framework existed.

### 2.5 Batch Rules

When receiving a batch of Tasks (multiple Task Prompts in a single prompt), execute sequentially. Complete each Task fully - execute, validate, and write the Task Log - before starting the next Task in the batch. Each Task gets its own Task Log at its specified `log_path`.

**Fail-fast:** If any Task results in Failed status, stop the batch. Do not proceed to remaining Tasks. After completing all Tasks (or stopping on failure), return a single batch result per `{GUIDE_PATH:task-logging}` §4.3 Batch Result Format. Do not defer logging to the end of the batch.

---

## 3. Task Execution Procedure

Sequential flow from Task Prompt receipt through completion. Task Validation and the Correction Loop form a cycle that repeats until success or a stop condition.

### 3.1 Task Prompt Receipt

On Task receipt, perform the following actions:
1. Check for batch envelope: if the prompt contains `batch: true` in frontmatter, it contains multiple Task Prompts separated by `---` delimiters. Execute each Task sequentially per §2.5 Batch Rules.
2. If Workspace section present: switch to the specified branch or worktree path before starting work.
3. If `has_dependencies: true`, continue to Context Integration, otherwise proceed to §3.3 Task Execution.

### 3.2 Context Integration

Perform the following actions:
1. Read the Context from Dependencies section.
2. Execute integration based on dependency type per §2.1 Context Integration Standards:
   - **Cross-dispatch:** Follow integration steps completely - read files, review artifacts, understand interfaces. Verify critical details by reading the key files referenced before proceeding.
   - **Intra-batch:** Use guidance to recall and build upon prior work; review referenced paths to refresh context if needed.
3. If integration issues discovered, apply decision rules from §2.1 Context Integration Standards.

### 3.3 Task Execution

Perform the following actions:
1. Execute Detailed Instructions sequentially, applying Guidance and relevant Rules from `{RULES_FILE}`, working toward the Objective.
2. When all instructions complete, communicate that implementation is complete and you are moving to validation. Continue to Task Validation.

### 3.4 Task Validation

Perform the following actions:
1. Execute autonomous checks from the Task Prompt's validation criteria per §2.2 Validation Standards: run tests, verify builds, confirm outputs exist and match expected structure. If any fail, continue to the correction loop. Ambiguous results: treat as failure and iterate; if iteration doesn't resolve, return Partial.
2. If criteria require User involvement: return Partial per §2.2 Validation Standards. Write the Task Log, include what was accomplished, what needs User review, and where deliverables are located.
3. If all criteria passed, proceed to §3.6 Task Completion with Success status.

### 3.5 Correction Loop

Perform the following actions:
1. Investigate the failure per §2.3 Iteration Standards: read error output, trace the cause, understand what went wrong.
2. Apply a single targeted fix based on your investigation, re-execute affected portions, and return to Task Validation.
3. If the correction does not resolve the issue after 2-3 attempts, stop per §2.3 Iteration Standards. Proceed to §3.6 Task Completion with Partial status and detailed diagnostics.

### 3.6 Task Completion

Perform the following actions:
1. Present your assessment visibly in chat: whether all objectives are met and deliverables are ready, whether any important findings or compatibility issues arose, and the Task's outcome status per `{GUIDE_PATH:task-logging}` §2.2 Outcome Standards.
2. Commit work to the assigned branch per §2.4 Version Control Standards.
3. Create Task Log per `{GUIDE_PATH:task-logging}` §3.1 Task Log Procedure at `log_path`.
4. Return a structured result summary per `{GUIDE_PATH:task-logging}` §3.2 Task Result Delivery as your final output.

---

## 4. Common Mistakes

- *Framework vocabulary in project output:* Commit messages, source comments, and code should describe the actual work - not the framework managing it. Never surface Task IDs, Step numbers, domain identifiers, or APM terminology in project-facing output.
- *Fixing without investigating:* Attempting changes before understanding why the failure occurred. Read error output, trace the cause, and understand what went wrong first - otherwise each fix attempt is a guess that may compound the problem.
- *Iterating past the limit:* Continuing to attempt corrections after 2-3 failed attempts. Each iteration consumes context budget and reduces reasoning quality. Return Partial with diagnostics - the Manager is better equipped to handle escalation with fresh context and full project awareness.
- *Working non-incrementally:* Writing large deliverables in one pass without testing intermediate results. Build incrementally - compile, run, or validate after each meaningful step rather than producing everything and then discovering issues.
- *Logging Success with incomplete validation:* Marking a Task as Success when validation criteria were not fully exercised. If criteria cannot be met (missing resources, need User cooperation), log as Partial and explain what remains rather than claiming Success with caveats.

---

**End of Guide**
