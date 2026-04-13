# Codex Trace Extraction

在用 Codex 评估较弱模型时，优先分析 session trace，而不是先看代码 diff。

你的目标是回答这些问题：

- 模型有没有真的读 skill / reference
- 模型有没有真的执行自己说要做的步骤
- tool_use 有没有缺失、失败或顺序错误
- 最终回答是否和执行轨迹一致

## 先把运行方式定死

做 skill 优化时，尽量把 Codex 运行参数固定，否则很难比较不同模型或不同任务。

一个硬规则：

- 优化器自己可以读用户提供的原始上下文目录
- 被测弱模型默认只能读隔离后的 skill 包，不应直接读取用户原始上下文目录、仓库或资料文件夹

也就是说，用户上下文主要用于“理解和蒸馏”，不是直接塞给弱模型做开卷考试。

本机 `codex exec --help` 已确认这些参数可用：

- `--model <model>`
- `--cd <path>`
- `--sandbox <read-only|workspace-write|danger-full-access>`
- `--skip-git-repo-check`
- `--json`
- `--output-last-message <file>`
- `--ephemeral`
- `codex exec resume <session_id>`

推荐基线命令：

```bash
codex exec \
  --cd /abs/path/to/isolation-workspace \
  --model openai/gpt-5.4 \
  --sandbox workspace-write \
  --skip-git-repo-check \
  --json \
  --output-last-message /abs/path/to/run-data/last-message.txt \
  "这里放任务提示词"
```

说明：

- `--cd` 很重要。benchmark 时应指向隔离目录，而不是用户原始上下文目录
- `--model` 要显式写，后面你要按真实模型名生成 `additional/<model>.md`
- `--sandbox` 尽量固定；做可比对基线时不要一会儿 `read-only`、一会儿 `workspace-write`
- `--json` 方便把 stdout 保存成事件流，和 session 文件互相对照
- `--output-last-message` 方便直接拿最终回答做回归比对
- 如果你需要 trace，就不要在基线 benchmark 里加 `--ephemeral`

如果你已经在一个外部隔离得足够严格的环境里，也可以再加：

```bash
--dangerously-bypass-approvals-and-sandbox
```

但这只适用于你确定外层环境已经安全兜底的情况。

## 如何做隔离目录

做 Codex benchmark 时，通常同时隔离两样东西：

- 工作目录
- `HOME`

一个实用做法是：

```bash
tmp_home="$(mktemp -d /tmp/skill-opt-codex-home.XXXXXX)"
tmp_workdir="$(mktemp -d /tmp/skill-opt-codex-work.XXXXXX)"
run_dir="$(mktemp -d /tmp/skill-opt-codex-run.XXXXXX)"

mkdir -p "$tmp_home/.codex/skills"
cp -R /abs/path/to/skill "$tmp_home/.codex/skills/"
```

这样做的目的有两个：

- skill 快照和 session 文件都落在同一份隔离 home 里
- 被测模型不会顺手读到你自己平时 `~/.codex/skills/` 里的其他 skill 或历史 session

更稳妥的做法是只复制 skill 包本身，包括：

- `SKILL.md`
- `reference/` 或 `references/`
- `additional/`
- skill 自带的 example / asset / script

不要把这些内容一起复制进去：

- 用户原始 repo
- 用户单独提供但尚未蒸馏进 skill 的资料目录
- 只有优化器自己需要看的临时分析笔记
- 与 benchmark 无关的工作区文件

如果你需要让弱模型使用“用户上下文里的某些事实”，正确做法通常不是把整个上下文目录暴露给它，而是把必要事实整理进 skill 的 reference / example，再重新跑 benchmark。

## 隔离后的运行示例

```bash
tmp_home="$(mktemp -d /tmp/skill-opt-codex-home.XXXXXX)"
tmp_workdir="$(mktemp -d /tmp/skill-opt-codex-work.XXXXXX)"
run_dir="$(mktemp -d /tmp/skill-opt-codex-run.XXXXXX)"

mkdir -p "$tmp_home/.codex/skills"
cp -R /abs/path/to/skill "$tmp_home/.codex/skills/"

HOME="$tmp_home" codex exec \
  --cd "$tmp_workdir" \
  --model openai/gpt-5.4 \
  --sandbox workspace-write \
  --skip-git-repo-check \
  --json \
  --output-last-message "$run_dir/last-message.txt" \
  "这里放任务提示词" \
  > "$run_dir/events.jsonl"
```

这样做的目的不是模拟真实开发环境，而是验证：这个 skill 自己是否足够完整，能不能单靠 skill 包把较弱模型带到正确路径上。

## 用户上下文目录怎么处理

很多 skill 不是“拿到一份 `SKILL.md` 就能独立工作”，而是强依赖用户额外提供的上下文目录，例如：

- 一个 repo
- 一个 examples 目录
- 一组模板、脚本、schema、配置文件
- 一个已经生成了一半的工程

所以在做弱模型评估前，先回答：

1. 用户让模型在哪个目录里工作
2. 那个目录里哪些文件是事实来源
3. skill 要求模型读哪些本地文件才算进入正确上下文
4. 失败是否来自模型没有进入对的目录、没有读关键文件，或者把目录内容理解错了

如果这些问题没答清楚，不要急着把失败写成“模型不聪明”。

但要把这个原则和隔离原则分开：

- 优化器自己需要读用户上下文，才能知道 skill 还缺什么
- 被测弱模型的 benchmark 默认不直接读这些上下文
- 如果 benchmark 一旦隔离就失败，优先怀疑 skill 没把必要上下文内化进去

## 先定位 session

Codex 的非 `--ephemeral` 运行会把 session 文件写到：

```text
$HOME/.codex/sessions/
```

基线 benchmark 最简单的做法是：

- 每个 run 用单独的隔离 `HOME`
- 直接看这个隔离 home 里最新生成的 `.jsonl` 文件

先看第一条 `session_meta`，拿到真正的 session id。然后重点看后续事件。

优先关注这些记录：

- `session_meta`
  看 `payload.id`
- `response_item`
  主要看 `message`、`reasoning`、`function_call`、`custom_tool_call`、`function_call_output`、`custom_tool_call_output`
- `event_msg`
  主要看 `token_count`，以及 `collab_agent_*` / `collab_waiting_*` 这类子 agent 事件

分析时优先回答：

- 有没有真正出现 `reasoning`
- 有没有真正发起工具调用
- 工具输出是否和后面的最终回答一致
- 是否出现了 subagent 但主线程没有等待或整合结果

## 续跑同一轮实验

如果你要在同一份 session 上继续实验，优先用：

```bash
HOME="$tmp_home" codex exec resume <session_id> \
  --model openai/gpt-5.4 \
  --json \
  --output-last-message "$run_dir/last-message.resume.txt" \
  "这里放新的补充提示词"
```

或者：

```bash
HOME="$tmp_home" codex exec resume --last \
  --model openai/gpt-5.4 \
  --json \
  --output-last-message "$run_dir/last-message.resume.txt" \
  "这里放新的补充提示词"
```

续跑时，保持同一个隔离 `HOME`，否则你会丢失原 session。

## 同时跑多个模型或多个任务

Codex 也可以并发启动多个 benchmark 进程。最重要的规则不是“怎么并发”，而是“不要把 run 混在一起”。

并发时的规则：

- 每个 run 都要单独记录 `model`、`task`、`workspace`、`HOME`、`session id`
- 每个 run 都要单独落 `events.jsonl` 和 `last-message.txt`
- 如果任务会改文件，优先给每个并发 run 单独的隔离 home / workspace
- 即使任务主要是读取和分析，也优先不要共享同一个 `HOME`，否则 session 文件会混在一起

## 分析 trace 时优先看什么

Codex benchmark 的 primary evidence 仍然是执行轨迹，不是 diff。

优先顺序建议是：

1. `response_item.reasoning`
2. `response_item.function_call` / `custom_tool_call`
3. 对应的 tool output
4. 最后的 assistant message
5. 最后才看文件改动

如果最终结果错了，但 trace 显示它连 skill / reference 都没读、或者决定要调工具却没调，那首先是过程失败，不是“答案差一点”。
