# handoff-bridge

`handoff-bridge` 解决的不是「记住更多」，而是「在手动 context 管理里，安全断开并恢复关键状态」。

它服务的场景很具体：

- 围绕一个 Git issue 做多步修复
- 一轮里工具调用很多，context 开始脏
- 需要 `/clear`，但下一轮还得知道做到哪一步、哪条路验过不通、下一步从哪儿接

## 它和 `/compact` 是分工，不是替代

| | `/compact` | `handoff-bridge` |
|---|---|---|
| 解决 | 同一思路里上下文太长 | 跨 session / 跨天的断点恢复 |
| 形态 | 压缩当前对话 | 显式落盘的最小状态包 |
| 时机 | 上下文还没明显腐败 | 工具噪音多 / 要 `/clear` / 换 session |
| 价值 | 连续性 | 可审查的断点 + 负面证据保留 |

负面证据是 handoff 最值钱的部分——已排除的方案、还在等确认的分歧、下次该跑的验证命令。这些靠 `compact` 一压就丢了。

## 它怎么分层

- **Git issue** — 问题定义、验收标准、讨论结论
- **代码 / diff / 测试输出** — 唯一的真相来源
- **`HANDOFF.md`** — 当前 issue 的动态状态：做到哪、试过哪些不通、下一步从哪儿接、悬念
- **`todolist.md`** — 当前 issue 的具体步骤和验证命令

handoff 只放指针和摘要，不复述事实。业务规则、协议字段、稳定约束走 `CLAUDE.md` / `.ruler/*` / `CLAUDE.local.md`，不进 HANDOFF。

## 推荐工作流

1. 用 Git issue 定义问题和验收。
2. 在 `todolist.md` 里维护这个 issue 的步骤和验证命令。
3. 要 `/clear` 或换 session 之前，写 `HANDOFF.md`——只保留动态状态、失败路径、下一步入口。
4. 新 session 开头先读 `HANDOFF.md` + `todolist.md`，等你点头再继续。

## 收尾长这样

```
我：今天到这儿了，handoff 一下

Claude：看了下——这个 issue 还有 in-progress，且上一次验证把指数退避方案排除了。准备更新 HANDOFF。

预览：
  ## last_session_id
  - active_issue: #142
  - branch: feat/oauth-migration
  - last_test: `pnpm test auth/callback`
  ...
  ## next_steps
  - 进行中：OAuth 回调签名验证（步骤 3/5）
  - 下一步：补 retry 用例 + 跑集成测试
  - 已排除：指数退避 2 次的方案（重试间隔太长，超时窗口对不上）

校验都过了。要 commit 吗?

我：commit，先别 push
```

它会先自检三件事：有没有未提交的工作状态、todolist 还有没有活、本轮有没有出现过失败路径或待确认项。三个都没中就直接告诉你 `无需 HANDOFF，可直接 clear`——不会硬塞一份空交接给你。

## 恢复长这样

```
我：恢复上次

Claude：上次状态：
- issue #142：OAuth 回调签名验证
- 进行中：验证签名逻辑（3/5）
- 下一步：补 retry 用例 + 跑集成测试
- 已排除：指数退避 2 次的方案

按这个继续?

我：继续
```

3-5 行就够，没点头之前它不会自己动手。

## 触发词

- 收尾：`结束这轮` / `交接` / `handoff`，或者准备 `/clear` 前
- 恢复：`恢复上次` / `继续昨天的` / `resume`，或者新会话第一句

## 安装

```bash
# 全局
git clone https://github.com/Gizele1/handoff-bridge.git ~/.claude/skills/handoff-bridge

# 只在当前项目用
git clone https://github.com/Gizele1/handoff-bridge.git .claude/skills/handoff-bridge
```

Claude Code 启动会自己扫到，不用额外配置。

## 接到新项目里要改

- `base_branch`：项目基线分支名，不一定叫 `main`
- `tz_offset`：默认时区
- `issue_number` / `issue_title`：issue 引用格式
- A4 黑名单：项目里"不该出现在 HANDOFF 的长期规则关键词"
- `last_test`：默认验证命令模板
- 提交规范：按项目自己的约定

## 不做的事

- 不改 `CLAUDE.md` / `.ruler/*` / `CLAUDE.local.md` 这类长期规则文件
- 不自动 `git commit`，只问
- 不在没有 live state 的时候硬写 handoff
- 不把业务规则、协议字段、长期约束塞进 `HANDOFF`
- 不把失败路径抹掉——那是 handoff 的核心价值
- 不把 `handoff` 当成 `/compact` 的替代品
- 不跟多 agent 内部的 stage handoff 混为一谈

---

MIT.
