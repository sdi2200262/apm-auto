# Contributing to APM Auto

Contributions are welcome. This is a custom APM adaptation for Claude Code - for contributing to the core APM framework, see the [official repository](https://github.com/sdi2200262/agentic-project-management).

## Ways to Contribute

- **Workflow testing and feedback** - Run APM Auto sessions on real projects and report issues
- **Template improvements** - Improve the commands, guides, agent definitions, and skills in `templates/`
- **Build system** - Placeholder additions, build pipeline enhancements
- **Documentation** - Improve the README, CONTRIBUTING, and other docs

## Repository Layout

- `templates/` - Source files that become the APM Auto installation (commands, guides, skills, agent definitions, artifact templates)
- `templates/_standards/` - Development-time specifications. Not included in builds.
- `build/` - Build system that processes templates into the Claude Code bundle
- `skills/apm-customization/` - Standalone skill for AI agents helping with further customization of this repository. Adapted for APM Auto's autonomous-dispatch model.

APM Auto installs via the official `agentic-pm` CLI; this repository does not ship its own CLI source.

## Making Changes

APM uses a top-down propagation model. All workflow changes follow this order:

1. **Update `templates/_standards/WORKFLOW.md` first** - the source of truth for all behavior
2. **Propagate to runtime files** - commands, guides, agent definitions, and skills implement the workflow spec

Changes that do not affect the workflow (wording adjustments, examples) can be made directly in runtime files.

Read the relevant `_standards/` files before making changes:
- `WORKFLOW.md` - formal workflow specification
- `TERMINOLOGY.md` - formal vocabulary
- `STRUCTURE.md` - structural standards for each file type
- `WRITING.md` - writing patterns and conventions

## Building and Testing

```bash
npm install
npm run build:release
```

This produces `dist/claude.zip` and `dist/apm-release.json`. Extract the bundle into a test project and verify changes work with Claude Code.

## Pull Requests

1. Create a feature branch
2. Make changes following the standards in `templates/_standards/`
3. Run `npm run build:release` to verify the build
4. Submit a PR with a clear description of what changed and why

## License

By contributing, you agree that your contributions will be licensed under MPL-2.0.

## Questions

- Technical questions: [GitHub Issues](https://github.com/sdi2200262/apm-auto/issues)
- General inquiries: reach out on Discord at `cobuter_man`
