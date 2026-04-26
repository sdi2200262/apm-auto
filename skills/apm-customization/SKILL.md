---
name: apm-customization
description: Guides an AI agent through customizing APM Auto templates, building releases, and managing this custom APM repository.
---

# APM Auto Customization Skill

## 1. Overview

**Reading Agent:** Any AI assistant working within a forked or templated APM Auto repository

This skill guides the customization of APM Auto templates. It assumes the Agent is operating within the APM Auto codebase itself (a fork or template of this repository) and can explore the repository structure directly.

APM Auto is a custom adaptation of the official [Agentic Project Management (APM)](https://github.com/sdi2200262/agentic-project-management) framework built for Claude Code. It replaces APM v1's user-mediated Worker model with autonomous subagent dispatch: the Manager spawns ephemeral subagents via Claude Code's `Agent()` tool to execute Tasks, reviews their output, merges branches, and continues without requiring the User to shuttle messages between chats. Customizations should preserve or thoughtfully extend this autonomous-dispatch model rather than reintroduce User mediation.

### 1.1 Objectives

- Navigate the APM Auto repository structure and understand what each part does
- Make targeted changes to templates, commands, guides, skills, or agent configurations
- Build and test changes locally
- Produce releases that can be installed via `apm custom`

### 1.2 How to Use

The User describes what they want to change or add. The Agent explores the relevant parts of the repository, proposes changes, and implements them after User approval. This skill provides the orientation needed to find the right files and understand how they connect.

---

## 2. Repository Structure

Explore the repository to understand the layout. The key directories are:

**`templates/`** - The source files that become the APM Auto installation. Everything the User receives when running `apm custom -r sdi2200262/apm-auto` originates here.

**`templates/commands/`** - Slash commands the User sends to the model. APM Auto reduces the v1 surface to five: initiate-planner, initiate-manager, handoff-manager, summarize-session, recover. Worker initiation and the Worker-side check-tasks/check-reports/handoff-worker commands are eliminated since Workers are ephemeral subagents rather than separate chats.

**`templates/guides/`** - Procedural files Agents read autonomously. Each guide contains a single procedure with operational standards, step-by-step actions, output specifications, and content guidelines.

**`templates/skills/`** - Shared capabilities read by multiple Agent roles. Each skill lives in its own directory with a `SKILL.md` file and optional supporting files.

**`templates/agents/`** - Subagent configurations. The `apm-worker` definition is the foundation of APM Auto's dispatch model - it is the system prompt the Manager attaches when spawning a Worker subagent via `Agent()`.

**`templates/apm/`** - Artifact templates that become the `.apm/` directory (Spec, Plan, Tracker, Memory Index templates).

**`templates/_standards/`** - Development-time specifications that define how templates should be written. These files are not included in builds. Read them to understand the design rules:
- `WORKFLOW.md` - The formal workflow specification, including APM Auto's autonomous-dispatch overlay (Manager coordination loop, dispatch-time dependency classification, subagent iteration limits, drift detection during review). This is the source of truth for all behavior.
- `TERMINOLOGY.md` - Formal vocabulary and defined concepts, including APM Auto terms (subagent dispatch, intra-batch and cross-dispatch dependencies, coordination loop)
- `STRUCTURE.md` - Structural standards for each file type
- `WRITING.md` - Writing patterns, tone, formatting

**`build/`** - The build system that processes templates into platform-specific bundles. APM Auto targets a single platform (Claude Code), so `build-config.json` defines one target.

**`skills/`** - Standalone skills (like this one) that are not part of the main APM Auto bundle. APM Auto installs via the official `agentic-pm` CLI and does not ship its own CLI source.

---

## 3. How Templates Become Installations

Templates use placeholders that the build system resolves at build time. Understanding this is essential for making changes.

Explore `build/processors/placeholders.js` to see all supported placeholders. Common ones include:

- `{VERSION}` - Release version
- `{RULES_FILE}` - `CLAUDE.md` for Claude Code
- `{SKILL_PATH:name}` - Resolved path to a skill file
- `{GUIDE_PATH:name}` - Resolved path to a guide file
- `{COMMAND_PATH:name}` - Resolved path to a command file
- `{ARGS}` - `$ARGUMENTS` for Claude Code's Markdown command format
- `{SUBAGENT_GUIDANCE}` - Claude Code's `Agent()` tool invocation guidance

Explore `build/build-config.json` to see the Claude Code target's directory layout, format, and platform-specific values.

**Building locally:**

```bash
npm install
npm run build:release
```

This produces a `dist/` directory with `claude.zip` and an `apm-release.json` manifest. The User can test locally by extracting the bundle into a project.

---

## 4. Making Changes

When the User requests a change, identify which layer it affects. All workflow changes follow a top-down propagation:

1. **Update `WORKFLOW.md` first** - Any change that affects APM Auto's behavior, procedures, or coordination patterns must be reflected in the workflow specification before modifying runtime files. `WORKFLOW.md` is the source of truth and includes both the inherited APM v1 procedures and APM Auto's autonomous-dispatch overlay.
2. **Propagate to runtime files** - Commands, guides, skills, and agent configurations implement the workflow spec. Update these to match the changes made in `WORKFLOW.md`, following the conventions in `STRUCTURE.md`, `WRITING.md`, and `TERMINOLOGY.md`.

Changes that do not affect the workflow (e.g. adjusting wording within existing procedures, adding examples to guidance fields) can be made directly in runtime files without updating `WORKFLOW.md`.

### Working with the Autonomous-Dispatch Model

APM Auto replaces v1's Worker-as-separate-chat model with Worker-as-subagent. When making changes, consider whether the change interacts with this model:

- Changes to Manager procedures should preserve the autonomous coordination loop (dispatch, review, merge, continue) and avoid reintroducing User-mediated message delivery
- Changes to Worker behavior happen in `templates/agents/apm-worker.md` (the subagent system prompt) and Task Prompt construction, not in standalone Worker commands
- Changes to dependency handling should preserve dispatch-time classification (intra-batch vs cross-dispatch) rather than planning-time classification
- Changes to debugging or error recovery should respect the no-nesting constraint (subagents cannot spawn subagents) and the iteration limit that prevents spiraling failures
- Changes to drift detection during review should preserve the Manager's ability to catch and correct subagent output before merging

### Template Content Changes

Most customizations involve modifying template files in `templates/`. The `_standards/` files define the conventions:

- Commands follow structural profiles defined in `STRUCTURE.md` (strict for initiation, lightweight for utility)
- Guides follow a five-section pattern (Overview, Operational Standards, Procedure, Structural Specifications, Content Guidelines)
- Skills have a required Overview section and free-form internal organization
- All files use the terminology defined in `TERMINOLOGY.md`
- Writing follows the patterns in `WRITING.md` (imperative mood, token efficiency, de-duplication)

Read the relevant `_standards/` file before making changes to understand the conventions in play.

### Adding New Files

When adding a new guide, skill, command, or agent:

1. Follow the structural conventions from `STRUCTURE.md` for that file type
2. Add YAML frontmatter where required (commands and skills require it, guides do not)
3. Use placeholders for any paths, rules file references, or platform-specific values
4. Update cross-references in other files if the new file should be loaded by an Agent

### Build Configuration Changes

If targeting additional platforms (deviating from APM Auto's Claude-Code-only design), modify `build/build-config.json` to add target definitions with directories, format, and platform-specific values. Note that the autonomous-dispatch model relies on Claude Code's `Agent()` tool - other platforms would require a different dispatch mechanism.

---

## 5. Releasing

After making changes, the User creates a release that can be installed via `apm custom`.

1. **Build** - Run `npm run build:release` to generate the bundle in `dist/`
2. **Test** - Extract the bundle into a test project and verify the changes work in Claude Code
3. **Tag** - Create a git tag following the versioning convention
4. **Release** - Create a GitHub Release and attach all files from `dist/` (the ZIP bundle and `apm-release.json`)

The `apm-release.json` manifest is what the CLI reads to discover available assistants in the release. Explore `build/generators/manifest.js` to understand its structure.

Users install from the custom repository with:

```bash
apm custom -r owner/repo
```

---

## 6. Communicating Changes

When a custom repository diverges further from APM Auto (or APM Auto itself diverges from the official APM release), changes should be documented:

- Update the repository's README to describe what was customized and why
- If the changes affect the workflow (new procedures, modified coordination patterns), note how the customization differs from the official documentation
- If adding new commands or skills, document their purpose and usage

Custom repositories carry trust implications for anyone who installs from them. Bundles can write files anywhere within the project directory, and the templates define how AI assistants behave. When publishing a custom repository, document what the customization changes so users can make informed trust decisions. If the customization adds files outside the standard assistant config directory (e.g., source files, configuration), note this explicitly in the README.

---

**End of Skill**
