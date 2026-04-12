# OpenCode Trace Extraction

在用 OpenCode 评估较弱模型时，优先分析 message trace，而不是先看代码 diff。

你的目标是回答这些问题：

- 模型有没有真的读 skill / reference
- 模型有没有真的执行自己说要做的步骤
- tool_use 有没有缺失、失败或顺序错误
- 最终回答是否和执行轨迹一致

## 先把运行方式定死

做 skill 优化时，尽量把 OpenCode 运行参数固定，否则很难比较不同模型或不同任务。

一个硬规则：

- 优化器自己可以读用户提供的原始上下文目录
- 被测弱模型默认只能读隔离后的 skill 包，不应直接读取用户原始上下文目录、仓库或资料文件夹

也就是说，用户上下文主要用于“理解和蒸馏”，不是直接塞给弱模型做开卷考试。

本机 `opencode run --help` 已确认这些参数可用：

- `--model provider/model`
- `--variant <effort>`
- `--dir <path>`
- `--format json`
- `--thinking`
- `--session`
- `--fork`
- `--title`

推荐基线命令：

```bash
opencode run \
  --dir /abs/path/to/isolation-dir \
  --model provider/model \
  --format json \
  --thinking \
  --title "benchmark: task-1 / provider-model" \
  "这里放任务提示词"
```

说明：

- `--dir` 很重要。benchmark 时应指向隔离目录，而不是用户原始上下文目录
- `--model` 要显式写，后面你要按真实模型名生成 `additional/<model>.md`
- `--variant` 只在 provider 支持时再加，例如 `high`、`max`、`minimal`
- `--format json --thinking` 让后续 trace 分析更稳定

如果你要先看可用模型：

```bash
opencode models
```

## 如何做隔离目录

基准运行前，先做一个临时目录，只放弱模型应该看到的 skill 内容。

最小做法：

```bash
tmp_dir="$(mktemp -d /tmp/skill-opt.XXXXXX)"
mkdir -p "$tmp_dir/.opencode/skills"
cp -R /abs/path/to/skill "$tmp_dir/.opencode/skills/"
```

如果 skill 目录名是 `my-skill`，那么弱模型实际看到的是：

```text
$tmp_dir/.opencode/skills/my-skill/
```

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
tmp_dir="$(mktemp -d /tmp/skill-opt.XXXXXX)"
mkdir -p "$tmp_dir/.opencode/skills"
cp -R /abs/path/to/skill "$tmp_dir/.opencode/skills/"

opencode run \
  --dir "$tmp_dir" \
  --model provider/model \
  --format json \
  --thinking \
  --title "task-1 / provider-model / isolated" \
  "这里放任务提示词"
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

优先用 CLI 找最近的 session，再按目录和时间缩小范围：

```bash
opencode session list --format json
```

选择规则：

- 优先选 `directory` 等于目标 workspace 的 session
- 优先选最近一次运行
- 如果任务有 fork / 子会话，不要只看根 session，要把整棵 session tree 一起看

如果你只想先筛“文件变更不大”的 run，可以查 `session` 表里的这些字段：

- `summary_additions`
- `summary_deletions`
- `summary_files`

它们只能用于缩小候选 session，不能单独拿来判断任务是否成功。

示例：

```bash
sqlite3 -json "$HOME/.local/share/opencode/opencode.db" "
select
  id,
  parent_id,
  title,
  directory,
  time_created,
  summary_additions,
  summary_deletions,
  summary_files
from session
order by time_created desc
limit 20;
"
```

如果你只想看低文件变更的 session，可以先用启发式阈值筛一轮，再手动确认：

```bash
sqlite3 -json "$HOME/.local/share/opencode/opencode.db" "
select
  id,
  title,
  directory,
  time_created,
  summary_additions,
  summary_deletions,
  summary_files
from session
where coalesce(summary_files, 0) <= 2
  and coalesce(summary_additions, 0) + coalesce(summary_deletions, 0) <= 30
order by time_created desc
limit 20;
"
```

上面的阈值只是筛选候选 run 的启发式，不是固定标准。

## 同时跑多个模型或多个任务

OpenCode 可以同时启动多个进程来做并发基准。最简单的方法是多终端、`tmux`、后台 job，或者任意你熟悉的进程管理方式。

同一个任务跑多个模型：

```bash
mkdir -p .skill-optimizer-runs

opencode run \
  --dir /abs/path/to/isolation-dir \
  --model openai/gpt-5.4 \
  --variant high \
  --format json \
  --thinking \
  --title "task-1 / gpt-5.4-high" \
  "这里放任务提示词" \
  > .skill-optimizer-runs/task-1.gpt-5.4-high.jsonl &

opencode run \
  --dir /abs/path/to/isolation-dir \
  --model anthropic/claude-opus-4.6 \
  --format json \
  --thinking \
  --title "task-1 / claude-opus-4.6" \
  "这里放任务提示词" \
  > .skill-optimizer-runs/task-1.claude-opus-4.6.jsonl &

wait
```

同一个模型跑多个任务：

```bash
opencode run \
  --dir /abs/path/to/isolation-dir \
  --model provider/model-a \
  --format json \
  --thinking \
  --title "task-1 / model-a" \
  "任务 1" \
  > .skill-optimizer-runs/task-1.model-a.jsonl &

opencode run \
  --dir /abs/path/to/isolation-dir \
  --model provider/model-a \
  --format json \
  --thinking \
  --title "task-2 / model-a" \
  "任务 2" \
  > .skill-optimizer-runs/task-2.model-a.jsonl &

wait
```

并发时的规则：

- 每个 run 都要单独记录 `model`、`task`、`directory`、`session id`
- 每个 run 都要单独落日志文件，不要混 stdout
- 想做可比对基线时，不要把多个 benchmark task 混进同一个 session
- 如果任务会改文件，优先给每个并发 run 单独的隔离副本、worktree 或沙箱目录
- 如果任务主要是读取和分析，多个 run 可以共享同一份只读 skill 副本，但不要共享用户原始上下文目录
- 如果你要从同一个历史 session 分叉实验，可以用 `--session <id> --fork`

不要把“并发运行”误解成“共享一个 session”。benchmark 需要的是多个独立 session，而不是同一个 session 里串多轮不同实验。

## 数据源怎么分工

OpenCode 的持久化主数据源是：

```text
~/.local/share/opencode/opencode.db
```

有些运行环境还会有 live session 目录：

```text
~/.opencode/sessions
```

但这个目录不一定存在，也不一定比数据库更完整。做结构化分析时，优先用 `opencode.db`。

表的职责：

- `session`：session id、title、directory、parent/child 关系、文件变更摘要
- `message`：每条消息的元数据，适合看 `role`、`modelID`、`providerID`、`finish`、token
- `part`：真正的事件流，是主要分析对象

一个关键点：

- `message.data` 往往没有最终文本内容，真正的 `thinking`、`text`、`tool` 基本都在 `part.data`
- 所以不要只查 `message` 就下结论

另一个关键点：

- `session.directory` 是判断模型是否真的处于正确用户上下文里的重要证据
- benchmark 时要确认 `session.directory` 指向的是隔离目录，而不是用户原始上下文目录
- 分析失败时要把 `session.directory`、隔离目录内容和目标 skill 快照对起来看

## 先看 `message` 元数据

先确认到底是哪一个模型、哪种 finish reason、token 大概多少：

```bash
sqlite3 -json "$HOME/.local/share/opencode/opencode.db" "
select
  m.id,
  m.session_id,
  m.time_created,
  json_extract(m.data, '$.role') as role,
  json_extract(m.data, '$.modelID') as model_id,
  json_extract(m.data, '$.providerID') as provider_id,
  json_extract(m.data, '$.finish') as finish,
  json_extract(m.data, '$.tokens.total') as total_tokens,
  json_extract(m.data, '$.tokens.input') as input_tokens,
  json_extract(m.data, '$.tokens.output') as output_tokens
from message m
where m.session_id = 'ses_xxx'
order by m.time_created asc, m.id asc;
"
```

看点：

- `role = assistant` 的消息有没有多轮
- `finish` 是 `stop` 还是 `tool-calls`
- 实际模型名是什么，后面写 `additional/<model>.md` 要用到

## 主要看 `part`

真正要分析的是 `part.data.type`。常见类型：

- `reasoning` / `thinking` / `thought`：模型思考
- `text`：用户或助手文本
- `tool`：工具调用轨迹
- `step-start` / `step-finish`：阶段边界，通常次要

优先把根 session 和所有子 session 一起拉出来：

```bash
sqlite3 -json "$HOME/.local/share/opencode/opencode.db" "
WITH RECURSIVE session_tree(id, parent_id, title, depth) AS (
  SELECT id, parent_id, title, 0
  FROM session
  WHERE id = 'ses_xxx'
  UNION ALL
  SELECT child.id, child.parent_id, child.title, parent.depth + 1
  FROM session child
  INNER JOIN session_tree parent ON child.parent_id = parent.id
)
SELECT
  p.id,
  p.time_created,
  p.session_id,
  session_tree.depth,
  json_extract(p.data, '$.type') as type,
  json_extract(p.data, '$.tool') as tool,
  json_extract(p.data, '$.state.status') as status,
  json_extract(p.data, '$.state.title') as title,
  substr(
    coalesce(
      json_extract(p.data, '$.text'),
      json_extract(p.data, '$.state.input.description'),
      json_extract(p.data, '$.state.error')
    ),
    1,
    500
  ) as content
FROM part p
INNER JOIN session_tree ON p.session_id = session_tree.id
WHERE json_extract(p.data, '$.type') IN ('reasoning', 'thinking', 'thought', 'text', 'tool')
ORDER BY p.time_created asc, p.id asc;
"
```

这个查询已经够你判断绝大多数执行问题。

## `tool` 事件重点看什么

`tool` 事件一般至少要看这些字段：

- `$.tool`
- `$.state.status`
- `$.state.title`
- `$.state.input.description`
- `$.state.error`

如果你需要进一步确认工具到底做了什么，再看：

- `$.state.input.command`
- `$.state.output`

但做 skill 优化时，不要一开始就沉进完整输出；优先看“为什么调用”“有没有调用”“调用是否失败”。

## 读 trace 的顺序

建议按这个顺序看：

1. 先看 session 是否选对
2. 再看 `message` 元数据，确认模型名和 finish reason
3. 再按时间顺序看 `part`
4. 先看 `reasoning` 和 `tool`
5. 最后再看最终 `text`

因为 skill 优化更关心过程失败，而不是只看最后一句话。

## 你要从 trace 里提炼什么

优先提炼这些模式：

- reasoning 里说“我要先读文档/先检查”，但没有对应 tool_use
- 已经有 tool_use，但缺关键一步，比如 build 了没 deploy，读了目录没读 reference
- 工具失败了，但最终回答假装成功
- 直接输出结论，没有任何验证工具
- 明明 skill 要求先读某个 reference，trace 里完全没有相关动作
- 子 session 偏航，主 session仍然照单全收

这些才是应该写进 `additional/<model>.md` 的内容来源。

## 不要怎么做

- 不要主要依赖代码 diff 判断模型是否理解了 skill
- 不要只看 `message`，因为内容经常在 `part`
- 不要只看根 session，fork / 子 session 经常藏着真正的失败点
- 不要把一次偶发 tool error 直接写成模型固有缺陷

## 一个最小判断模板

分析一次 run 时，至少回答：

1. 这是哪个模型跑的
2. 它有没有读到关键上下文
3. 它有没有把关键工具真的执行出来
4. 它在哪一步偏离了 skill
5. 这个偏离是主 skill 不清楚，还是某个模型特有弱点
