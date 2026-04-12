# Skill Optimizer

[简体中文](./README.zh-CN.md)

`skill-optimizer` is a small repository for a Codex/OpenCode skill that helps stronger models improve how weaker models execute a user-provided skill.

The core idea is:

- a strong model studies the target skill and any user-provided context
- weak models are benchmarked in isolation, using only the skill package
- failures are diagnosed from OpenCode message traces, especially thinking and tool-use events
- model-specific fixes are written back into `additional/<model>.md`
- shared fixes are pushed into the main skill or references

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
            └── opencode.md
```

## What The Skill Covers

- designing benchmark tasks for a target skill
- identifying where weaker models skip reading references or skip required tools
- isolating weak-model runs so they cannot directly access the user's raw context
- running OpenCode with explicit models and variants
- running multiple OpenCode benchmarks in parallel
- extracting evidence from OpenCode `session`, `message`, and `part` records
- turning repeated failure patterns into model-specific `additional/<model>.md` guidance

## Current Skill Files

- [skills/skill-optimizer/SKILL.md](./skills/skill-optimizer/SKILL.md): main workflow for optimizing a skill on weaker models
- [skills/skill-optimizer/reference/opencode.md](./skills/skill-optimizer/reference/opencode.md): OpenCode execution, isolation, and trace-analysis reference

## Intended Workflow

1. Read the target skill and understand the user's extra context.
2. Distill any context that should become part of the skill itself.
3. Create isolated benchmark environments that only contain the skill package.
4. Run weaker models through OpenCode with explicit `--model`, `--dir`, `--format json`, and `--thinking`.
5. Inspect traces before code diffs.
6. Add model-specific constraints to `additional/<model>.md` when needed.
7. Move shared fixes back into the main skill or references.

## License

This repository is licensed under the MIT License. See [LICENSE](./LICENSE).
