# Skill Optimizer

[English](./README.md)

`skill-optimizer` 是一个面向 Codex / OpenCode 的 skill 仓库，用来帮助更强的模型优化“用户提供的 skill”在较弱模型上的执行表现。

核心思路是：

- 先由强模型理解目标 skill 和用户提供的额外上下文
- 再把较弱模型放进隔离环境，只让它看到 skill 包本身
- 通过 OpenCode 的 message trace，尤其是 thinking 和 tool-use 事件，分析失败原因
- 把模型特有的失败模式写回 `additional/<model>.md`
- 把所有模型共享的问题回收到主 skill 或 reference

`reference/` 目录用于存放不同 agent 框架的使用方式。当前只有 `opencode.md`，后续可以继续增加 `openclaw`、`codex`、`claude code` 等框架的参考文档。

## 安装方式

使用 `npx skills add` 安装：

```bash
npx skills add https://github.com/igniscloud/skill-optimizer.git
```

如果你的环境更适合 SSH，也可以替换成对应的仓库地址。

## 仓库结构

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

## 这个 Skill 解决什么问题

- 为目标 skill 设计基准任务
- 找出较弱模型在哪些地方漏读 reference、漏调工具、提前收工
- 把弱模型运行隔离开，避免它直接读取用户原始上下文
- 用显式模型参数和 variant 运行 OpenCode
- 并发跑多个模型或多个任务
- 从 OpenCode 的 `session`、`message`、`part` 记录中提取证据
- 把稳定复现的失败模式沉淀成 `additional/<model>.md`

## 当前关键文件

- [skills/skill-optimizer/SKILL.md](./skills/skill-optimizer/SKILL.md)：优化弱模型执行表现的主流程
- [skills/skill-optimizer/reference/opencode.md](./skills/skill-optimizer/reference/opencode.md)：当前 OpenCode 专用的运行、隔离和 trace 分析参考

## 预期工作流

1. 阅读目标 skill，并理解用户提供的额外上下文。
2. 把本来应该内化到 skill 的上下文信息蒸馏进去。
3. 创建只包含 skill 包的隔离 benchmark 环境。
4. 通过 OpenCode 用显式的 `--model`、`--dir`、`--format json`、`--thinking` 来跑较弱模型。
5. 优先分析 trace，而不是先看代码 diff。
6. 需要时把模型特有约束写入 `additional/<model>.md`。
7. 把共性修复回收到主 skill 或 reference。

## 许可证

本仓库使用 MIT License。见 [LICENSE](./LICENSE)。
