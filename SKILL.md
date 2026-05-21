---
name: handoff-bridge
description: 以 issue 为单位的跨 session handoff skill：在准备 /clear 或恢复新会话时，轻量检查是否值得写入 HANDOFF.md + todolist.md，只保留 active_issue、已验证的失败路径、下一步入口和验收命令。Use when the user says /clear, 清理会话, 结束这轮, 交接, handoff, 恢复上次, 继续昨天的, resume, or starts a new session that needs prior context restored.
---

# handoff-bridge

`handoff-bridge` 解决的不是“记住更多”，而是“在手动 context 管理里，安全断开并恢复关键状态”。

它服务于这种工作流：

- 围绕单个 Git issue 做多步修复
- 一轮里工具调用很多，context 开始脏
- 需要 `/clear`，但不能丢掉下一轮必须知道的东西

开干前先告诉用户一句：`I'm using the handoff-bridge skill to {收尾|恢复} this session.`

## 定位

- `HANDOFF.md` 只记录跨 session 仍然有用的动态状态。
- `todolist.md` 只记录当前 issue 的步骤和验收。
- Git issue 负责问题定义、讨论结论、验收标准。
- 代码、diff、测试输出是事实来源；handoff 只放指针和摘要。

`HANDOFF.md` 和 `todolist.md` 的边界口诀：

- **抽象层不同**：`todolist` 是操作层（原子步骤 + 验收命令），`HANDOFF` 是上下文层（位置坐标 + 指针 + 失败路径）。
- **失败路径只入 HANDOFF**：从 `todolist` 划掉/删掉的已排除方案，必须搬进 `HANDOFF.next_steps.已排除`，否则下一轮会重复试错。
- **HANDOFF 不复述步骤**：`todolist` 的 5 条原子步骤在 `HANDOFF` 里只对应一句"步骤 3/5"。HANDOFF 涨到 80 行就是这条没守住。

## 最重要的一条

`HANDOFF.md` 不是日志，也不是聊天纪要。它只保留下次恢复时还会影响判断的内容：当前做到哪一步、试过但失败了什么、下一步从哪里接、还有什么没拍板。

业务规则、协议字段、长期约束、实现细节里的稳定事实，都应该进项目长期规则文件（`CLAUDE.md` / `.ruler/*` / `CLAUDE.local.md` 之类），不要塞进 HANDOFF。否则这个文件很快会退化成一锅杂物。

## 和 `/compact` 的分工

- `compact` 适合同一条思路还在继续、上下文还没明显腐败的时候。
- `handoff-bridge` 适合要跨 session、跨天，或者工具噪音已经很多的 issue 修复。
- `compact` 负责连续性，`handoff` 负责可审查的断点恢复。
- `handoff` 特别要保留负面证据：哪些路已经验证不通，避免下一轮重复试错。

## 什么时候触发

`/clear`、"清理会话"、"结束这轮"、"交接"、"handoff"、"恢复上次"、"继续昨天的"、"resume" → 先检查是否存在需要保留的动态状态，再决定写不写。

轻量判断只看这些信号：

1. 当前 issue 还没结束，且 `todolist.md` 里还有 `pending` / `in-progress` / `blocked`。
2. 已经验证过的失败路径、被否决方案、用户拍板前的关键分歧还没写下来。
3. 这轮要跨 `/clear`，下次恢复不看 HANDOFF 就会重新猜。
4. 用户明确说要 handoff / resume / 继续上次。

如果这些信号都没有，就直接告诉用户：`无需 HANDOFF，可直接 clear。`

session 还在跑、用户没有要切断上下文 —— 不要主动写 handoff。

## 优先级和分工

```
项目 CLAUDE.md > 项目长期规则文件（.ruler/*、CLAUDE.local.md 等） > HANDOFF.md > todolist.md > MEMORY.md
```

| 文件 | 装什么 | 生命周期 |
|---|---|---|
| 项目长期规则文件 | 业务规则、协议字段、禁令、稳定实现约束 | 永久 |
| Git issue | 问题定义、验收标准、讨论结论 | 跨 session |
| `HANDOFF.md` | 当前 issue 的动态状态、失败路径、下一步入口 | 跨 session |
| `todolist.md` | 当前 issue 的步骤、验收命令、阻塞项 | 当前回合 |
| `MEMORY.md` 等长期画像 | 跨项目仍有效的决策指针 | 长期 |

`HANDOFF` 和 `todolist` 都会出现"进行中 / 下一步 / 阻塞"这类字眼，含义不同别混：

| 字段 | 在 `todolist.md` 里 | 在 `HANDOFF.md` 里 |
|---|---|---|
| 进行中 | 某一行的状态标签（`状态: in-progress`） | 整个 issue 的位置坐标（"步骤 3/5，正在补 X 单测"） |
| 下一步 | 待执行的下一条具体命令 | 下次 session 的入口指针（粒度更粗） |
| 阻塞 | 某步卡住的原因（行级） | issue 级的悬而未决项，常带"等用户/外部 X" |

---

## 流程 A — 收尾

大路径：`A1 自检 → A2 写预览 → A3 同步 todolist → A4 校验 → A5 问 commit`。

### A1. 轻量自检

任一命中就继续：

```bash
# 1. 当前有值得保留的工作状态
git status --porcelain | grep -v '^??' | head -1

# 2. todolist 里还有活项
grep -E '状态.*:\s*(pending|in-progress|blocked)' todolist.md | head -1

# 3. 本轮有需要保留的失败路径/待确认项
# 例如：blocked、waiting、rejected、need-confirm、failed-path
```

三个都没中 → 告诉用户“无需 HANDOFF，可直接 clear”。

### A2. 写更新预览（先给用户看，别直接落盘）

注意一点：`HANDOFF.md` 始终只有一套标题。不要追加第二套，只改段里的内容。

文件不在就按这个 schema 创建；在就保留标题，刷新段内容：

```markdown
## last_session_id

- session_id: {YYYY-MM-DD}-{slug}
- ended_at: {YYYY-MM-DDTHH:MM:SS{tz_offset}}
- branch: {当前分支}
- base_branch: {项目基线分支}
- active_issue: #{issue_number}
- issue_title: {issue 标题}
- auto_session_summary: {本轮 ≤2 行要点}
- last_test: `{最近一次验证命令}`

## decisions

- **{YYYY-MM-DD}** {跨会话仍生效的决策要点}。详见 `{相对路径或锚点}`。

## next_steps

- 进行中：{任务 + 当前步骤}
- 下一步：{下一个可执行动作}
- 已排除：{已经验证不通、下一轮不要再重复试的路径}
- 阻塞：{等用户或外部什么，没有就省略}

## 上次 session 末尾留下的悬念（≤5 行）

- {YYYY-MM-DD HH:MM} — {1-2 行说明 + 等什么}
```

具体怎么改：

- `last_session_id` 整段换掉，写本轮的。
- `decisions` 只留会跨 session 继续有效的决策。
- `next_steps` 整段换掉，写下次入口，尤其保留 `已排除`。
- 本轮从 `todolist.md` 删掉/划掉的"试过但不通"的方案，**必须**搬进 `next_steps.已排除` 并补一句原因。todolist 不留这条，HANDOFF 也不留 = 下一轮 Claude 会重新提议同一条死路。
- 悬念区最多 5 条，超过就删最老的。

### A3. 顺手把 `todolist.md` 同步一下

- 干完的勾掉（改 `done`）
- 卡住的标 `blocked` 并补一句原因
- 本轮真正要继续做的步骤保留在前面
- 和 issue 无关的杂项不要留在 todolist 里
- 如果全 `done` 了 → 只留前 4 行 schema 注释，其余清空

### A4. 校验

```bash
grep -cE '^## last_session_id'            HANDOFF.md   # =1
grep -cE '^## decisions'                  HANDOFF.md   # =1
grep -cE '^## next_steps'                 HANDOFF.md   # =1
grep -cE '^## 上次 session 末尾留下的悬念' HANDOFF.md   # =1

wc -l < HANDOFF.md                                     # <= 80

# 按项目自定义再加黑名单
# grep -iE '{业务关键词1}|{业务关键词2}' HANDOFF.md

grep -E 'session_id:\s*[0-9]{4}-[0-9]{2}-[0-9]{2}' HANDOFF.md
```

按项目自定义的黑名单只应该拦截长期规则词，不要把 issue 号、分支名、测试命令误判掉。

哪条没过 → 给用户看失败项和修法 → 修完再跑一遍 → 都过了再到 A5。

### A5. 问要不要 commit

校验过了就问一句：

> `HANDOFF.md / todolist.md` 已经更新并通过校验。是否 commit & push？（按本项目提交规范）

如果项目用了规则合成工具，而本轮又动了规则源，提醒用户跑一下同步命令。没用这类工具的项目就当没说。

---

## 流程 B — 恢复

路径：`B1 读两份文件 → B2 写恢复报告 → 等用户确认 → 按 next_steps 接着干`。
要改方向，先把 `HANDOFF` 的 `next_steps` 改掉再动手。

### B1. 读

```
Read HANDOFF.md
Read todolist.md
```

### B2. 报告，3-5 行就够

格式固定：

```
基于 HANDOFF + todolist 的上次状态：
- issue：#{issue_number} {issue_title}
- 进行中：{next_steps.进行中}
- 下一步：{next_steps.下一步}
- 阻塞（如有）：{next_steps.阻塞}
- 已排除（如有）：{next_steps.已排除 或 悬念区最近 1 条}

按此路径继续？
```

用户没确认之前别动手。要改方向，先改 `next_steps`。

---

## 速查

| 看到什么 | 怎么做 |
|---|---|
| `/clear` / "清理会话" / "结束这轮" | 流程 A |
| "交接" / "handoff" | 流程 A |
| 新会话 / "恢复上次" / "继续昨天的" / "resume" | 流程 B |
| session 还在跑 | 不管 |
| A1 三信号都没中 | 告诉用户没必要 HANDOFF，可以 clear |
| A4 没过 | 不能到 A5，先让用户修 |

## 几件不做的事

- 不改项目长期规则文件，要改用户自己来
- 不自动 `git commit`，只问
- 不主动跑规则合成命令，只提醒
- 不把业务规则 / 协议字段 / 长期约束写进 `HANDOFF`
- 不把失败路径抹掉；它们是 handoff 的价值之一
- 不在 session 还活着、也没有切断需求的时候强行写 handoff
- 不把 `handoff` 当成 `compact` 的替代品；两者分工不同
- 不跟多 agent 内部的 stage handoff 搞混

## 接到新项目里要自定义的

1. **`base_branch`** — 项目基线分支名（main / master / develop ...）
2. **`tz_offset`** — 时区（+08:00 / +00:00 / -05:00 ...）
3. **`issue_number` / `issue_title`** — issue 的引用方式，按你的工具链统一
4. **A4 黑名单** — 项目里“绝对不该出现在 HANDOFF 的长期规则关键词”
5. **`last_test`** — 默认的验证命令模板
6. **A5 提交规范** — 改成项目自己的 commit 约定
7. **规则合成工具** — 没用的话就删掉相关提醒
