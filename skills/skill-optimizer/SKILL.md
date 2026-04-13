---
name: skill-optimizer
description: 当用户要优化一个已有 skill 在较弱模型或 OpenCode / Codex 这类 agent 框架里运行时的表现时使用，适用于基准任务设计、失败复现、message/thinking/tool_use 分析，以及为特定模型补 `additional/<model>.md`。
---

# Skill Optimizer

当任务是“优化一个已有 skill，让较弱模型更稳定地执行它”时使用。

你的目标不是重写整个 skill，而是减少较弱模型的这些问题：

- 漏读主 `SKILL.md` 或关键 reference
- 应该调用工具却没有调用
- 说自己会做，但没有真正执行
- 用猜测代替查证
- 太早结束，没有做关键验证

默认角色分工：

- 优化器本身尽量使用当前可用的强模型
- 被测对象使用用户指定的较弱模型，通常通过 OpenCode 或 Codex 跑
- 优化器可以读取用户提供的额外上下文；被测对象默认不应直接读取这些原始上下文

## 默认选择规则

- 如果用户明确指定了 agent 框架，按用户指定执行
- 如果用户没有指定 agent 框架，默认优先用 subagent 的方式执行 benchmark / 复现 / 回归
- 如果用户明确指定了模型，按用户指定执行
- 如果用户没有指定模型，默认使用当前模型，而不是额外猜一个“较弱模型”
- 当你改用默认 subagent 路径时，仍然要保持隔离思路：优化器掌握原始上下文，被测对象只拿到你允许它看到的 skill 包或蒸馏后的最小上下文

这里的“较弱模型”不是只看排行榜，而是看它是否会稳定地：

- 漏步骤
- 漏读 skill / reference
- 漏工具调用
- 在没有验证时提前收工

## 先做什么

1. 先读用户提供的 skill 目录和额外上下文。
   - 这里的额外上下文通常不只是几句话，还可能是一个强相关文件夹、仓库或资料目录
   - 先搞清楚这个 skill 默认依赖哪些本地事实来源：目录结构、模板文件、reference、脚本、示例、配置、生成物
   - 如果不理解这些上下文，你就很难判断弱模型到底是“智力不足”，还是“根本没拿到足够上下文”
   - 但这些原始上下文主要是给优化器自己理解问题，不是默认直接暴露给被测弱模型
2. 找出这个 skill 成功执行时必须满足的最小条件：
   - 必须读哪些文件
   - 必须调用哪些工具
   - 哪些步骤不能靠记忆或猜测
   - 哪些用户上下文必须被蒸馏进 skill，本来不该依赖运行时临时读取
3. 如果用户没有给评测任务，就自己设计 3-5 个基准任务，至少覆盖：
   - 一个最短 happy path
   - 一个必须读 reference 才能做对的任务
   - 一个必须调用关键工具才不会错的任务
   - 一个容易让弱模型提前结束或脑补的任务
4. 任务要像真实用户请求，不要把标准答案直接写进任务里。

## 输出物

默认输出物是这些：

- 在目标 skill 目录下新增 `additional/`
- 为需要补强的模型写 `additional/<model>.md`
- 必要时补充或收紧主 `SKILL.md`
- 必要时把“必须先理解哪些用户提供上下文”写回主 `SKILL.md`
- 必要时把用户上下文中的关键信息蒸馏进 skill 的 reference / example，而不是要求弱模型临场再读原目录
- 给出基准任务、失败模式、修复方式和验证结果

## `reference/` 目录约定

- `reference/` 用于放不同 agent 框架的使用方式和运行/trace 分析说明
- 当前实现了 `reference/opencode.md` 和 `reference/codex.md`
- 后续可以继续增加如 `reference/openclaw.md`、`reference/claude-code.md`
- 当你使用某个具体 agent 框架做 benchmark 时，优先读取对应的 framework reference，而不是把所有框架规则都塞进主 `SKILL.md`

## `additional/` 目录约定

- 目录位置：`<skill-root>/additional/`
- 文件名：`<model>.md`
- 模型名优先使用运行时实际使用的 model id；如果文件名不安全，再做最小归一化
- 文件内容只写“这个模型特别容易犯的错”以及“遇到这些情况时的硬约束动作”
- 不要把主 `SKILL.md` 大段复制进去

每个 `additional/<model>.md` 应该尽量短、硬、可执行。建议结构：

```md
# <model>

## 常见失败
- ...

## 强制动作
- ...

## 结束前检查
- ...
```

写法要求：

- 写成可执行指令，不要写泛泛评价
- 只写稳定复现的失败，不要围绕一次偶发问题过拟合
- 如果所有模型都会犯同样的错，优先改主 `SKILL.md` 或 reference，而不是只写模型补丁

## 优化流程

1. 基线运行
   - 优化器先完整理解用户提供的 skill 和额外上下文
   - 然后把被测弱模型放进隔离环境，只给它 skill 包本身，不给原始用户上下文目录
   - 如果用户没指定 agent 框架，默认先用 subagent 跑这一步；只有在用户明确要求或任务上下文明确要求时，才切到 OpenCode / Codex
   - 如果用户没指定模型，默认沿用当前模型做这一步
   - 用较弱模型跑基准任务
   - 每次运行都记下使用的模型名和 session id
   - 每次运行都记下对应的隔离目录，以及它是从哪份 skill 快照生成的
   - 如果是 OpenCode，优先按 `reference/opencode.md` 的方式取 trace
   - 如果是 Codex，优先按 `reference/codex.md` 的方式取 trace
2. 过程分析
   - 不要先看代码 diff；优先看 message、thinking、tool_use
   - 先确认模型是否只靠 skill 包就能完成任务，而不是偷读原始用户上下文
   - 检查模型有没有真的读 skill / reference、有没有真的执行自己说要做的事
   - 失败先记录成“过程失败”，最终结果只作为辅助证据
3. 提炼失败模式
   - 失败模式必须可复用、可描述、可约束
   - 常见类型包括：漏读 reference、漏关键工具、tool_use 顺序错误、reasoning 里决定了但没有执行、直接宣布完成而没有验证、把建议当成硬规则、把硬规则当成建议、skill 里缺少本该内化的上下文
4. 写回 skill
   - 模型特有问题：写进 `additional/<model>.md`
   - 所有模型共性问题：改主 `SKILL.md`
   - 缺少事实来源：补 reference 或 example，不要只加空泛说明
   - 如果某个关键信息只存在于用户原始目录里，而弱模型又不该直接读取那个目录，就把它蒸馏进 skill
5. 回归验证
   - 用同一批任务重新跑
   - 回归时继续保持隔离，不要因为方便而把用户原始上下文重新暴露给弱模型
   - 只有在结果确实更稳时，才保留新增规则
   - 还没解决的失败要明确记录，不要假装已经完成

## 何时看代码

- 代码或 diff 只能作为 secondary evidence
- primary evidence 永远是执行轨迹：模型读了什么、想了什么、调用了什么工具、在哪一步停止
- 只有在你已经通过 trace 定位到失败点后，才去看代码确认影响

## 使用 OpenCode 时的规则

- 如果你控制 OpenCode 调用参数，优先使用隔离目录里的 `opencode run --format json --thinking`
- 用指定模型运行时，优先显式传 `--model`，需要时再传 `--variant`
- 当你要同时测试多个模型或多个任务时，可以并发启动多个 OpenCode 进程；具体命令和隔离规则见 `reference/opencode.md`
- 分析 trace 时优先看 `part` 里的 `reasoning` / `thinking` / `thought` / `tool` / `text`
- `message` 更适合看模型名、provider、token、finish reason；不要把它当成主要内容来源
- 需要 session 选择、递归取子 session、或者筛选低文件变更 run 时，读 `reference/opencode.md`

## 使用 Codex 时的规则

- 如果你控制 Codex 调用参数，优先使用 `codex exec --json`
- 用指定模型运行时，优先显式传 `--model`
- benchmark 默认不要加 `--ephemeral`，否则 session 不会落盘，后续没法稳定分析 trace
- benchmark 时优先给 Codex 一个隔离后的 `HOME` 和工作目录；具体命令与隔离规则见 `reference/codex.md`
- 分析 trace 时优先看 Codex session 里的 `response_item` 和 `event_msg`，不要先看最终 diff
- 需要继续同一轮实验时，优先用 `codex exec resume <session_id>` 或 `codex exec resume --last`
- 当你要同时测试多个模型或多个任务时，可以并发启动多个 Codex 进程，但优先给每个 run 单独的隔离 home / workspace

## 使用 Subagent 时的规则

- 只有在用户没有指定 agent 框架时，subagent 才是默认路径；如果用户指定了 OpenCode 或 Codex，不要擅自改回 subagent
- 如果用户没有指定模型，subagent 默认使用当前模型
- subagent 任务要明确写清：它只能使用 skill 包本身，还是还能读取哪些蒸馏后的上下文
- 不要把优化器自己读过的整份原始上下文直接转发给 subagent；只传 benchmark 真正需要的最小信息
- 分析 subagent 失败时，优先看它有没有真的读 `SKILL.md`、有没有真的调用该调的工具、有没有提前宣布完成

## 交付时必须说明

- 用了哪些基准任务
- 哪些失败是稳定复现的
- 新增了哪些 `additional/<model>.md`
- 哪些规则被放回主 `SKILL.md`
- 哪些风险还没被完全压住
