# Skill Optimizer

[简体中文](./README.zh-CN.md)

`skill-optimizer` is a small repository for a Codex/OpenCode skill that helps stronger models improve how weaker models execute a user-provided skill.

The core idea is:

- a strong model studies the target skill and any user-provided context
- if the user does not specify an agent framework, the optimizer defaults to a subagent execution path
- if the user does not specify a model, the optimizer defaults to the current model
- weak models are benchmarked in isolation, using only the skill package
- failures are diagnosed from Codex/OpenCode session traces, especially thinking and tool-use events
- model-specific fixes are written back into `additional/<model>.md`
- shared fixes are pushed into the main skill or references

The `reference/` directory is reserved for agent-framework-specific usage guides. It currently contains both `opencode.md` and `codex.md`. Later it can also include references for frameworks such as `openclaw` or `claude code`.

## Installation

Install the skill with `npx skills add`:

```bash
npx skills add https://github.com/igniscloud/skill-optimizer.git
```

If your environment prefers SSH, replace the repository URL accordingly.

## Repository Layout

```text
.
├── LICENSE
├── README.md
├── README.zh-CN.md
└── skills/
    └── skill-optimizer/
        ├── SKILL.md
        └── reference/
            ├── codex.md
            └── opencode.md
```

## What The Skill Covers

- designing benchmark tasks for a target skill
- defaulting to subagent execution when the user does not name an agent framework
- defaulting to the current model when the user does not name a model
- identifying where weaker models skip reading references or skip required tools
- isolating weak-model runs so they cannot directly access the user's raw context
- running Codex and OpenCode with explicit models and reproducible execution settings
- running multiple Codex/OpenCode benchmarks in parallel
- extracting evidence from framework trace records before looking at code diffs
- turning repeated failure patterns into model-specific `additional/<model>.md` guidance

## Current Skill Files

- [skills/skill-optimizer/SKILL.md](./skills/skill-optimizer/SKILL.md): main workflow for optimizing a skill on weaker models
- [skills/skill-optimizer/reference/codex.md](./skills/skill-optimizer/reference/codex.md): Codex-specific execution, isolation, resume, and trace-analysis reference
- [skills/skill-optimizer/reference/opencode.md](./skills/skill-optimizer/reference/opencode.md): current OpenCode-specific execution, isolation, and trace-analysis reference

## Intended Workflow

1. Read the target skill and understand the user's extra context.
2. If the user did not specify a framework, default to subagent execution; if the user did not specify a model, default to the current model.
3. Distill any context that should become part of the skill itself.
4. Create isolated benchmark environments that only contain the skill package.
5. Run weaker models through subagents, Codex, or OpenCode with explicit execution settings and isolated skill snapshots.
6. Inspect traces before code diffs.
7. Add model-specific constraints to `additional/<model>.md` when needed.
8. Move shared fixes back into the main skill or references.

## License

This repository is licensed under the MIT License. See [LICENSE](./LICENSE).
