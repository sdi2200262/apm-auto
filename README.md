# APM Auto

[![License: MPL-2.0](https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg)](https://opensource.org/licenses/MPL-2.0)

*A custom APM adaptation for Claude Code with autonomous subagent dispatch.*

## What is APM Auto?

APM Auto is a custom adaptation of [Agentic Project Management (APM)](https://github.com/sdi2200262/agentic-project-management) built for Claude Code. It replaces APM's user-mediated Worker model with Manager-driven subagent dispatch - the Manager autonomously spawns ephemeral subagents via Claude Code's `Agent()` tool to execute Tasks, reviews their output, and continues without requiring the User to shuttle messages between chat windows.

It is designed for prototyping, fast execution, and simpler projects where the overhead of managing separate Worker chats is not worth it. Think vibe-coding with structure - you plan with the Planner, approve the plan, and then the Manager runs through the implementation autonomously, only pausing when it genuinely needs your input.

## Installation

Navigate to your project directory, initialize through the `agentic-pm CLI`,

```bash
npm install -g agentic-pm
apm custom -r sdi2200262/apm-auto
```

Open Claude Code and start planning:

```
/apm-1-initiate-planner
```

The Planner collaborates with you through project discovery and creates the planning documents. Once approved, open a new chat and run `/apm-2-initiate-manager` - the Manager takes over and runs the implementation autonomously.

## How It Works

APM Auto coordinates two agent roles:

- **Planner** - Conducts structured project discovery and produces three planning documents: Spec (what to build), Plan (how work is organized), and Rules (how work is performed).
- **Manager** - Autonomously dispatches subagents to execute Tasks, reviews their results, merges branches, and maintains project state. Only pauses for genuine human judgment.

Subagents are spawned using a custom `apm-worker` agent definition that provides behavioral rules as the system prompt. Each subagent is ephemeral - it receives a self-contained Task Prompt, executes, writes a Task Log, and returns a structured result.

## Trade-offs

APM Auto trades visibility and control for speed and autonomy:

| | APM Auto | Official APM |
|---|---|---|
| **Speed** | Faster - no message shuttling between chats | Slower - User mediates every exchange |
| **Visibility** | Lower - subagent work is only visible after completion via Task Log and code changes | Higher - User sees every Worker interaction in real time |
| **Steering** | Minimal - User can only steer through the Manager, not mid-task | Full - User can guide Workers during execution |
| **Context control** | Less - subagents start fresh every time, no accumulated domain familiarity | More - Workers build context across tasks, can be handed off |
| **Debugging** | Subagents cannot spawn debug subagents (no nesting) - limited to 2-3 correction attempts before returning to Manager | Workers can spawn debug subagents for deeper investigation |
| **Platform** | Claude Code only | 6 platforms supported |
| **Best for** | Prototypes, fast iterations, simpler projects, vibe-coding | Complex projects, multi-domain work, projects requiring close oversight |

The Manager compensates for reduced visibility through detailed Task Logs, drift detection during review, explicit scope boundaries in Task Prompts, and a hard iteration limit that prevents subagents from spiraling through failed debugging attempts.

## Commands

| # | Command | Agent | Purpose |
|---|---------|-------|---------|
| 1 | `/apm-1-initiate-planner` | Planner | Planning Phase |
| 2 | `/apm-2-initiate-manager` | Manager | Implementation Phase (autonomous coordination) |
| 3 | `/apm-3-handoff-manager` | Manager | Transfer context to fresh Manager instance |
| 4 | `/apm-4-summarize-session` | Standalone | Session summary and archival |
| 5 | `/apm-5-recover` | Manager | Reconstruct context after compaction |

## Documentation

For the full APM workflow documentation, see [agentic-project-management.dev](https://agentic-project-management.dev). This adaptation shares APM's core concepts (planning documents, Stages, Tasks, Memory, Handoff) with the dispatch model adapted for autonomous subagent execution.

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines. Report bugs or suggest features via [GitHub Issues](https://github.com/sdi2200262/apm-auto/issues).

## License

Licensed under the **Mozilla Public License 2.0 (MPL-2.0)**. See [LICENSE](LICENSE) for full details.

Based on [Agentic Project Management](https://github.com/sdi2200262/agentic-project-management) by [CobuterMan](https://github.com/sdi2200262).
