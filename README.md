# handoff-bridge

> 跨 Claude Code session 的"收尾 + 恢复"skill。让多天、多次 `/clear` 的工作流不再丢上下文。

## Why

Claude Code 的单次 session 上下文是有限的——长任务跑几天总要 `/clear`，每次 `clear` 之后新 session 就像失忆。常见的痛点：

- 上次走到哪一步？哪个 PR 在等审？哪个决策是已拍板的？要翻 git log 和 chat 历史拼凑
- 项目里的长期规则（CLAUDE.md / `.ruler/*` 等）和当下的临时状态混在一起，越攒越乱
- 收尾随手记几句，下次自己都看不懂

`handoff-bridge` 把这件事变成两个清晰流程：

- **收尾**（流程 A）：自动判定是否需要写交接、按固定 schema 生成增量、做软校验（结构 / 体积 / 黑名单）、再提示 commit
- **恢复**（流程 B）：新会话第一条自动读 `HANDOFF.md` + `todolist.md`，3–5 行恢复报告，确认后继续

核心约束：**HANDOFF 只放跨会话仍生效的动态状态**（任务定位 / 悬念 / 下次入口）。业务规则和长期约束走项目规则文件，黑名单 grep 会在收尾时拦截误写。

## 安装

把仓库克隆到下列任一位置：

```bash
# 用户级（全局可用）
git clone https://github.com/Gizele1/handoff-bridge.git ~/.claude/skills/handoff-bridge

# 项目级（仅当前项目可用，推荐）
git clone https://github.com/Gizele1/handoff-bridge.git .claude/skills/handoff-bridge
```

Claude Code 启动时会自动识别 `SKILL.md` 的 frontmatter，无需额外配置。

## 怎么用

直接对 Claude Code 说下面任意一句即可触发：

| 想做什么 | 说什么 |
|---|---|
| 收尾本轮、准备 `/clear` | `结束这轮` / `交接` / `handoff` |
| 新会话恢复上次状态 | `恢复上次` / `继续昨天的` / `resume` |

Claude 会按 `SKILL.md` 里的两条流程走，全程跟你确认，不会自作主张提交或改长期规则。

## 自定义

skill 是通用模板，首次接入新项目时按需调以下几处（详见 `SKILL.md` 末尾的"自定义建议"）：

- 基线分支名（`main` / `master` / `develop` ...）
- 时区偏移
- A4 黑名单关键词（你项目里"绝对不该进入 HANDOFF 的长期规则关键词"）
- 提交规范（Conventional Commits / 自定义格式）

## 设计取舍

- **软校验而非硬阻塞**：所有 grep 检查在收尾时跑，失败就提示用户修，不会卡死流程
- **不自动 commit**：仅询问，按你项目的提交规范走
- **不碰长期规则**：本 skill 只写 `HANDOFF.md` 和 `todolist.md`，不动 `CLAUDE.md` / `.ruler/*` 等长期规则文件
- **不和 multi-agent stage handoff 混用**：那是 agent 间的中间产物，和跨 session 是两件事

## License

MIT — 详见 [`LICENSE`](LICENSE)。
