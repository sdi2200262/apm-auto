# APM Auto Standalone Skills

Optional skills that live outside the main APM Auto bundle. These are used on demand for specific situations rather than installed by `apm custom`.

## Available Skills

### apm-customization

Guides an AI agent through customizing APM Auto templates, navigating the repository structure, making changes, building, and releasing. No installation needed - this skill is already present in any fork or template of this repository. The AI agent reads it directly from `skills/apm-customization/SKILL.md` when working within the repo. If your platform does not discover it automatically, copy it to the platform's skills directory (e.g., `.claude/skills/apm-customization/SKILL.md`).

The skill is adapted for APM Auto specifically - it covers the autonomous-dispatch overlay (Manager coordination loop, subagent dispatch via `Agent()`, dispatch-time dependency classification, drift detection during review) on top of the standard APM customization workflow.

## Contributing

To propose a new standalone skill, open an issue or pull request on the [APM Auto repository](https://github.com/sdi2200262/apm-auto).
