# Know Each Other

让不同 AI Agent 互相感知对方在做什么。不依赖中心调度服务，不依赖特定 Agent 平台。**文件系统就是协议层。**

## 一句话

Hermes 分配任务、Claude Code 认领干活。两个 Agent 通过 `.collab/` 目录下的 state.md（日志）+ board.md（公告栏）交接。你一句话触发——"同步项目"、"加需求"、"更新 wiki"。

## 思考历程

### 起点：多 Agent 真的能互相知道吗？

Claude Code 有 Agent Teams——多个实例通过共享任务 JSON + 邮箱文件通信。底层全是文件系统，没有中心调度进程。Codex CLI 和 OpenCode 则是零。

### 想深一层：跨平台呢？

Hermes 和 Claude Code 是不同平台的 Agent。怎么让它们协作？答案是：**让他们读写同一个文件。**

```
Hermes 写需求 → REQUIREMENTS.md → Claude Code 读 → 开始干活
Claude Code 完成后 → 写状态 → Hermes 读 → 更新 wiki
```

### 落地：2 个文件就够了

最初的方案有 3 个文件 + 模板 + ⚠️ 信号系统 + 冲突处理规则。太重。

精简到两个：

| 文件 | 作用 | 谁写 |
|------|------|------|
| `board.md` | 任务公告栏——谁分活、谁领活、做到哪了 | Hermes 分配，Claude Code 认领+改状态 |
| `state.md` | 运行日志——时间线上每个人干了什么 | 双方追加 |

### 关键设计决策

**需求写一句话，不写详细规格。** Claude Code 的编码判断力优于 Hermes。给目标不给代码。省 token，发挥 Agent 能力。

**不追求自动触发。** LLM Agent 无法被"强制"执行任何操作。用户一句话触发最可靠——"同步项目"、"加需求到 blog：XXX"。

**不依赖特定平台。** 目录名叫 `.collab/`，不叫 `.hermes/` 或 `.claude/`。任何能读文件的 Agent 都能参与。

## 使用方式

### 1. 安装

```bash
# Hermes skill：复制 SKILL.md 到 ~/.hermes/skills/know-each-other/
# 或直接加载仓库
```

### 2. 初始化项目

在 Hermes 中说：

> 初始化 know-each-other

Hermes 创建 `.collab/` 目录和两个文件。

### 3. Claude Code 配置

把以下内容放在项目 `CLAUDE.md` 第一条：

```markdown
## 🔴 协作协议（know-each-other）

1. 每轮开始前读 .collab/board.md，认领 ⏳ 任务 → 标记 🔄
2. 任务开始/结束时 → 追加 .collab/state.md
3. 改动影响其他模块时 → ⚠️ 前缀标注
4. 完成任务 → board.md 移到 ✅，标注 commit hash
```

### 4. 日常命令

| 你说 | Hermes 做 |
|------|----------|
| "同步项目" | 读 state + board → 汇报状态 |
| "加需求到 blog：文章搜索" | 追加 board ⏳ + 写日志 |
| "更新 wiki" | 读日志 → 写 wiki → 移动完成项 |
| "更新 CLAUDE.md" | 读日志 → 追加式修改 CLAUDE.md |

## 文件格式

### board.md（任务公告栏）

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

### state.md（运行日志）

```markdown
## 2026-07-28

[14:30] Hermes — 新增需求 REQ-005（文章搜索）
[15:00] Claude Code — 完成 REQ-003（登录 API），commit a1b2c3d
[15:45] Claude Code — 🔄 开始 REQ-004（前端登录页），涉及 /login/page.tsx
[16:20] Claude Code — ⚠️ 修改了 API 返回格式：{code,data,msg} → {status,payload}
[16:45] Claude Code — 完成 REQ-004，commit e5f6g7h
```

## 设计原则

- **文件系统即协议层** — 不需要 API、不需要数据库、不需要后台进程
- **人类可读** — Markdown 纯文本，可 cat、可 grep、可手动编辑
- **Agent 无关** — 不绑定 Hermes、不绑定 Claude Code
- **用户触发** — 不追求自动，你一句话最可靠
- **简洁优先** — 2 个文件，需求一句话，不给代码

## 许可

MIT
