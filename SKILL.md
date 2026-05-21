---
name: handoff-bridge
description: 跨 Claude Code session 的状态交接器——会话末把任务进度/悬念/下次入口写进 HANDOFF.md + todolist.md，新会话开头读它俩给一份 3-5 行恢复摘要。Use whenever the user says /clear, 清理会话, 结束这轮, 交接, handoff, 恢复上次, 继续昨天的, resume, or starts a new session that needs prior context restored. Prefer triggering even when the user only hints at wrapping up or resuming.
---

# handoff-bridge

跨 Claude Code session 做两件事：

- **收尾**：判断要不要写交接，更新 `HANDOFF.md`，跑一遍 grep 校验，问你要不要 commit。
- **恢复**：读 `HANDOFF.md` + `todolist.md`，给你一份 3-5 行的状态摘要，等你确认再继续。

开干前先告诉用户一句：`I'm using the handoff-bridge skill to {收尾|恢复} this session.`

## 最重要的一条

`HANDOFF.md` 里**只放跨会话仍生效的动态状态**——任务在哪一步、什么悬念没解、下次从哪儿继续。业务规则、协议字段、长期约束这些得走项目长期规则文件（`CLAUDE.md` / `.ruler/*` 之类），千万别往 HANDOFF 里塞。混进去文件就废了，A4 的 grep 黑名单是兜底的最后一道。

## 什么时候触发

`/clear`、"清理会话"、"结束这轮"、"交接"、"handoff" → 走流程 A。
新会话第一条，或者用户说"恢复上次"、"继续昨天的"、"resume" → 走流程 B。

session 还在跑、用户没说要中断 —— 别管，正常干活。
流程 A 的 A1 三个信号一个都没中 —— 告诉用户"没必要写 HANDOFF，直接 clear 就行"然后退。
业务规则 / 协议字段 / 禁令条款 —— 那是长期规则文件的事，不写 HANDOFF。

## 优先级和分工

冲突时按这个顺序看：

```
项目 CLAUDE.md > 项目长期规则文件（.ruler/*、CLAUDE.local.md 等） > HANDOFF.md > 长期用户画像（MEMORY.md 等）
```

| 文件 | 装什么 | 生命周期 |
|---|---|---|
| 项目长期规则文件（`.ruler/*`、`CLAUDE.local.md` 等） | 业务规则、协议字段、禁令 | 永久 |
| `HANDOFF.md` | 跨 session 的任务定位、悬念、下次入口 | 固定 4 段标题，收尾时改段内，体积 ≤80 行 |
| `todolist.md` | 当轮步骤 + 验收命令 | 收尾后清空，留前 4 行 schema 注释 |
| `MEMORY.md` 等长期画像 | 跨项目仍有效的决策指针 | 长期 |

---

## 流程 A — 收尾

大路径：`A1 自检 → A2 写预览 → A3 同步 todolist → A4 校验 → A5 问 commit`。
A1 三信号一个都没中就直接退。A4 没过就让用户修，修完再校验，过了才到 A5。

### A1. 三信号自检

任一命中就继续：

```bash
# 1. 有未提交改动
git status --porcelain | grep -v '^??' | head -1    # 非空即命中

# 2. todolist 还有活
grep -E '状态.*:\s*(pending|in-progress|blocked)' todolist.md | head -1

# 3. 最近 5 轮对话出现过悬念关键词
# 关键词：待批准 / 阻塞 / 等用户确认 / 需要你决定 / 待确认 / approval gate / blocked / waiting
```

三个全没中 → 告诉用户"无需 HANDOFF，可直接 clear"，退出。

### A2. 写更新预览（先给用户看，别直接落盘）

注意一点：`HANDOFF.md` 始终只有一套 4 段标题。不要追加第二套——只改段里的内容。

文件不在就照下面创建；在就保留标题，改段内内容。占位符按项目实际填：

```markdown
## last_session_id

- session_id: {YYYY-MM-DD}-{slug}             # slug 是 3-5 字短语
- ended_at: {YYYY-MM-DDTHH:MM:SS{tz_offset}}
- branch: {当前分支}
- base_branch: {项目基线分支}                  # 常见 main / master / develop
- auto_session_summary: {本轮 ≤2 行要点}

## decisions

# 只记跨会话仍生效的决策。每条 ≤2 行，详情放个文件指针就行
- **{YYYY-MM-DD}** {决策要点}。详见 `{相对路径或锚点}`。

## next_steps

# 和 todolist.md 同步，相当于下次会话的入口
- 进行中：{任务 + 当前步骤}
- 下一步：{下一个可执行动作}
- 阻塞：{等用户或外部什么，没有就省了}

## 上次 session 末尾留下的悬念（≤5 行）

- {YYYY-MM-DD HH:MM} — {1-2 行说明 + 等什么}
```

具体怎么改：

- `last_session_id` 整段换掉，写本轮的。
- `decisions` 加新的跨会话决策；过期或者已经搬到长期规则文件里的删掉。
- `next_steps` 整段换掉，写下次入口。
- 悬念区追加本轮没解的，超过 5 条砍最老的。

填好之后大概长这样（参考，别照抄字段值）：

```markdown
## last_session_id

- session_id: 2026-05-21-oauth-refactor
- ended_at: 2026-05-21T18:42:10+08:00
- branch: feat/oauth-migration
- base_branch: main
- auto_session_summary: 拆完 OAuth 回调签名验证 3/5；retry 上限待用户拍板。

## decisions

- **2026-05-19** access_token 改用 RS256，公私钥走 KMS。详见 `docs/auth/keys.md`。
- **2026-05-21** 回调失败统一走 `AuthCallbackError`，不再抛裸 `Error`。详见 `src/auth/callback.ts:42`。

## next_steps

- 进行中：OAuth 回调签名验证（步骤 3/5，正在补 `verifySignature` 单测）
- 下一步：补 retry 用例 + 跑 `pnpm test:integration auth/`
- 阻塞：retry 上限 3 还是 5，等用户定

## 上次 session 末尾留下的悬念（≤5 行）

- 2026-05-21 18:30 — retry 上限定 3 还是 5，影响 `MAX_RETRY` 常量和集成测试 fixture。
```

### A3. 顺手把 todolist.md 同步一下

- 干完的勾掉（改 `done`）
- 卡住的标 `blocked` 并补一句原因
- 本轮新冒出来的复杂任务（>3 步）加进去，补验收表格
- 如果全 `done` 了 → 只留前 4 行 schema 注释，其余清空

### A4. 校验

```bash
# 4 个段标题各自只能有一份
grep -cE '^## last_session_id'                     HANDOFF.md   # =1
grep -cE '^## decisions'                           HANDOFF.md   # =1
grep -cE '^## next_steps'                          HANDOFF.md   # =1
grep -cE '^## 上次 session 末尾留下的悬念'          HANDOFF.md   # =1

# 总行数
wc -l < HANDOFF.md                                              # ≤80，超了就精简最老条目

# 黑名单（按项目自定义，下面是占位）
# 比如协议字段、命令字、加密参数名之类
# grep -iE '{业务关键词1}|{业务关键词2}' HANDOFF.md             # 期望空

# session_id 格式
grep -E 'session_id:\s*[0-9]{4}-[0-9]{2}-[0-9]{2}' HANDOFF.md   # ≥1
```

哪条没过 → 给用户看失败项和怎么修 → 修完再跑一遍 → 都过了再到 A5。

> 黑名单这一项每个项目都不一样。IoT 项目可能是某些蓝牙协议字段名，支付项目可能是密钥字段名。把这些塞进 A4 grep 列表，收尾时就能挡住误写。

### A5. 问要不要 commit

校验过了就问一句：

> "HANDOFF.md / todolist.md 已经更新并通过校验。是否 commit & push？（按本项目提交规范）"

如果项目用了 ruler 之类的规则合成工具，而本轮又动了规则源，提醒用户跑一下同步命令（比如 `ruler apply --agents codex,claude,gemini`）。没用这类工具的项目这条就当没说。

---

## 流程 B — 恢复

路径：`B1 读两份文件 → B2 写恢复报告 → 等用户确认 → 按 next_steps 接着干`。
要改方向，先把 HANDOFF 的 `next_steps` 改掉再动手。

### B1. 读

```
Read HANDOFF.md   # 全量
Read todolist.md  # 全量
```

### B2. 报告，3-5 行就够

格式固定：

```
基于 HANDOFF + todolist 的上次状态：
- 上次 session：{last_session_id.session_id} @ {branch}
- 进行中：{next_steps.进行中}
- 下一步：{next_steps.下一步}
- 阻塞（如有）：{next_steps.阻塞}
- 悬念（如有）：{悬念区最近 1 条}

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

不改项目长期规则文件，要改用户自己来。不自动 `git commit`，只问。不主动跑规则合成命令，只提醒。不让业务规则 / 协议字段进 HANDOFF（A4 黑名单兜着）。也别跟多 agent 内部的 stage handoff 搞混——那是 agent 之间传东西，跟跨 session 不是一码事。

## 接到新项目里要自定义的

1. **`base_branch`** — 项目基线分支名（main / master / develop ...）
2. **`tz_offset`** — 时区（+08:00 / +00:00 / -05:00 ...）
3. **A4 黑名单** — 项目里"绝对不该出现在 HANDOFF 的长期规则关键词"，比如协议字段名、密钥字段名、约束条款 ID
4. **A5 提交规范** — 改成项目自己的 commit 约定（Conventional Commits / `[type]subject` / Gitmoji ...）
5. **规则合成工具** — 没用 ruler 之类的，把 A5 末尾那段删掉
