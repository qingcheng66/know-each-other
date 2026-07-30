---
name: know-each-other
description: 多 Agent 协作协议——通过共享文件（.collab/）实现 Hermes 与 Claude Code 之间的任务分配、状态同步和日志记录。用户一句话触发，无需手动管理文件。
triggers:
  - 同步项目|handoff|项目状态
  - 加需求|新增需求|添加任务
  - 更新 wiki|同步 wiki
  - 更新 CLAUDE.md|维护 CLAUDE
---

# Know Each Other — 多 Agent 协作协议

让不同 AI Agent 通过共享文件互相感知对方在做什么。不依赖中心调度服务，不依赖特定 Agent 平台。

## 核心原则

> Agent 之间不需要 API。文件系统就是协议层。

Hermes 分配任务、维护需求、记录日志。Claude Code 认领任务、执行开发、汇报进度。两个 Agent 通过 `.collab/` 目录下的两个文件交接。

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│ Hermes Agent │      │   .collab/    │      │ Claude Code  │
│  分配任务 ────┼─────→│   board.md   │←─────┼── 认领任务   │
│  写日志     ──┼─────→│   state.md   │←─────┼── 汇报进度   │
└─────────────┘      └──────────────┘      └─────────────┘
```

## 文件结构

```
{项目根目录}/.collab/
├── state.md    ← 运行日志（双方追加）
└── board.md    ← 任务公告栏（Hermes 发布，Claude Code 认领）
```

## 用户命令 → Hermes 行为

| 用户说 | Hermes 自动做 |
|--------|-------------|
| "同步项目" / "handoff" | ① 确认项目目录 → ② 读 state.md 最近 50 行 → ③ 读 board.md → ④ 汇报：谁做了什么、什么在进行、有无 ⚠️、下一步建议 |
| "加需求到 {项目}：XXX" | ① 读 board.md → ② 追加一行到 ⏳ 区 → ③ 追加 state.md 日志 `[时间] Hermes — 新增需求 XXX` |
| "更新 wiki" | ① 读 state.md 了解最近改动 → ② 更新 wiki → ③ board.md 里完成项移到 ✅ 区 → ④ 追加 state.md |
| "更新 CLAUDE.md" | ① 读 state.md（最近的决策/坑）→ ② 读现有 CLAUDE.md → ③ 追加式修改 |

## state.md 格式

```markdown
## YYYY-MM-DD

[HH:MM] Hermes — 做什么/决策
[HH:MM] Claude Code — 做什么，commit xxxxxx
[HH:MM] Claude Code — 🔄 开始 {任务}，涉及 {文件}
[HH:MM] Claude Code — ⚠️ 修改了 {影响其他模块的改动}
```

- `🔄` 表示任务开始
- `⚠️` 表示改动可能影响其他 Agent / 模块
- 每条一行，简洁可 grep

## board.md 格式

```markdown
# 任务公告栏

> Hermes 分配 → Claude Code 认领 → 做完移 ✅
> 状态：⏳ 待认领 | 🔄 进行中 | ✅ 完成 | ❌ 放弃

## 🔄 进行中

- REQ-004 前端登录页面 (Claude Code, 15:45)

## ⏳ 待认领

- REQ-005 文章搜索功能
- REQ-006 关于页 GitHub 贡献图

## ✅ 最近完成

- REQ-003 登录 API (Claude Code, a1b2c3d, 7/28)
```

需求条目：一句话摘要。不写详细规格——留给 Claude Code 发挥。

## Claude Code 的 CLAUDE.md 规则

把以下内容放在项目 CLAUDE.md 的第一条：

```markdown
## 🔴 协作协议（know-each-other）

1. 每轮开始前读 .collab/board.md，认领 ⏳ 任务 → 标记 🔄
2. 任务开始/结束时 → 追加 .collab/state.md
3. 改动影响其他模块时 → ⚠️ 前缀标注
4. 完成任务 → board.md 移到 ✅，标注 commit hash
```

## 初始化

用户说"初始化 know-each-other"时：

1. 确认项目根目录
2. 创建 `.collab/` 目录 + `state.md` + `board.md`
3. 追加 state.md 首条日志
4. 提示用户把 CLAUDE.md 规则复制到项目

---

## 设计决策

- **不要自动每轮触发** — Agent 无法被强制。用户一句话触发最可靠
- **不要详细需求规格** — 省 token，发挥 Claude Code 编码能力
- **不要外部守护进程** — 文件系统就是全部基础设施
- **不要数据库** — Markdown 文件可读可编辑可 grep
- **不绑定 Agent 平台** — `.collab/` 目录与 Hermes/Claude Code 解耦
