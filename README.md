# handoff-bridge

每次 `/clear` 之后 Claude 就失忆。跨好几天的活儿做完一轮，下次回来不知道走到哪了——上次哪个 PR 在等审、retry 上限定了没、有没有半截的迁移，全靠翻 git log 和 chat history 拼。烦了一阵之后弄出来了这个。

思路挺朴素：用 `HANDOFF.md` 加 `todolist.md` 兜住跨 session 的状态。收尾的时候 Claude 顺手把它们写一下，下次进来第一句话先把它们读一遍。

## 收尾长这样

```
我：今天到这儿了，handoff 一下

Claude：看了下——有未提交改动 + todolist 还剩两个 in-progress，准备更新 HANDOFF。

预览（last_session_id 和 next_steps 整段换，decisions 加一条）：
  ## last_session_id
  - session_id: 2026-05-21-oauth-refactor
  - branch: feat/oauth-migration
  ...
  ## next_steps
  - 进行中：OAuth 回调签名验证（步骤 3/5）
  - 下一步：补 retry 用例 + 跑集成测试

校验都过了。要 commit 吗?

我：commit，先别 push
```

它会自检三件事：有没有未提交改动、todolist 还有没有活、最近几轮对话有没有"等我确认"这类悬念。三个都没中就直接告诉你"没啥要交接的"，省得你看一堆空话。

## 恢复长这样

```
我：恢复上次

Claude：上次状态:
- session 2026-05-21-oauth-refactor，分支 feat/oauth-migration
- 进行中：OAuth 回调签名验证 3/5
- 下一步：补 retry 用例 + 跑集成测试
- 悬念：retry 上限你还没定，3 还是 5

按这个继续?

我：定 3，继续
```

3-5 行的摘要，没点头之前它不会自己动手。

## 怎么触发

收尾："结束这轮"、"交接"、"handoff"，或者 `/clear` 之前都行。
恢复："恢复上次"、"继续昨天的"、"resume"，或者新会话第一句直接问。

## 装

```bash
# 全局
git clone https://github.com/Gizele1/handoff-bridge.git ~/.claude/skills/handoff-bridge

# 只在当前项目用
git clone https://github.com/Gizele1/handoff-bridge.git .claude/skills/handoff-bridge
```

Claude Code 启动会自己扫到，不用配置。

## 接到新项目里大概要改这几处

`SKILL.md` 末尾写得细，这里只提一下：基线分支名（不一定叫 main）、时区（默认按 +08:00 写的）、commit 规范。

要紧的是 A4 那个 grep 黑名单——把项目里"不该出现在 HANDOFF 的长期规则关键词"列进去。IoT 项目可能是蓝牙协议字段名，支付项目可能是密钥字段名。这步漏了之后它就会顺着对话漂到 HANDOFF 里，越积越乱，最后整个文件没法看。

## 一些不做的事

不动 `CLAUDE.md` 和 `.ruler/*` 之类的长期规则文件——要改自己改。不替你 commit，只问。还有跟多 agent 内部那种 stage handoff 不是一回事，别搞混了。

---

MIT.
