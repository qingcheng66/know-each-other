# Know Each Other

让不同 AI Agent 互相感知对方在做什么。不依赖中心调度服务，不依赖特定 Agent 平台。**文件系统就是协议层。**

## 一句话

Hermes 分配任务、Claude Code 认领干活。两个 Agent 通过 `.collab/` 目录下的 state.md（日志）+ board.md（公告栏）交接。你一句话触发——"同步项目"、"加需求"、"更新 wiki"。

**v2.0 新增：支持多个 Agent 会话并行开发**（文件域锁 + 会话注册表 + 认领原子化），详见下文。

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

精简到两个核心文件：

| 文件 | 作用 | 谁写 |
|------|------|------|
| `board.md` | 任务公告栏——谁分活、谁领活、做到哪了 | Hermes 分配，Claude Code 认领+改状态 |
| `state.md` | 运行日志——时间线上每个人干了什么 | 双方追加 |

### v2.0：多 Agent 并行怎么办？（2026-08-01）

实际使用中同时开多个 Agent 会话并行开发时发现竞争风险，任何多 Agent 共享一个仓库都会遇到：

- **board 认领竞态**：多个任务被同一时间戳全部标记 🔄，无会话归属
- **文件冲突热点**：两个任务同时改同一个共享文件（如全局样式、根布局），谁后保存谁覆盖谁；全仓级任务（如 lint 清理）与任何并行任务都可能撞
- **git 提交互相卷入**：两个会话各自 commit/push，第二个 push 被拒（non-fast-forward）
- **.collab 无锁**：两会话同时写 state.md / board.md，Markdown 无锁，后者覆盖前者且不可见

**教训**：给任务标"归属"（agent-a 做这个、agent-b 做那个）只是计划，LLM 会话不保证遵守。所以 v2.0 用**机制强制**替代计划约定：

| 机制 | 实现 | 原理 |
|------|------|------|
| 文件域锁 | `.collab/locks/` | `mkdir` 是原子操作，失败即被占，锁失败直接挡在门外 |
| 会话注册表 | `.collab/sessions/` | 启动/结束必须注册，谁活着、占着什么域一目了然 |
| 认领原子化 | board 变更跟 commit | git 会拒绝非快进提交，冲突可见可解决 |

### 关键设计决策

**需求写一句话，不写详细规格。** 执行 Agent 的编码判断力优于 Hermes。给目标不给代码。省 token，发挥 Agent 能力。

**不追求自动触发。** LLM Agent 无法被"强制"执行任何操作。用户一句话触发最可靠——"同步项目"、"加需求到 {项目}：XXX"。

**不依赖特定平台。** 目录名叫 `.collab/`，不叫 `.hermes/` 或 `.claude/`。任何能读文件的 Agent 都能参与。

**锁用 mkdir 不用 touch。** mkdir 原子、失败即冲突；touch 文件会静默覆盖。

**协议文档通用化，项目细节外置。** skill 不写死具体项目的文件路径/任务编号，文件域清单在初始化时按项目定义。

## 使用方式

### 1. 安装

```bash
# Hermes skill：复制 SKILL.md 到 ~/.hermes/skills/know-each-other/
# 或直接加载本仓库
```

### 2. 初始化项目

在 Hermes 中说：

> 初始化 know-each-other

Hermes 创建 `.collab/` 目录 + state.md + board.md + locks/ + sessions/，并按项目结构定义文件域清单。

### 3. Claude Code 配置

把以下内容放在项目 `CLAUDE.md` 第一条（文件域清单按该项目结构替换）：

```markdown
## 🔴 协作协议（know-each-other v2.0）

0. 启动：读 .collab/sessions/ + locks/，写自己的注册文件 .collab/sessions/<会话名>.md
1. 读 .collab/board.md 认领 ⏳ 任务 → 标记 🔄 + 会话名，board 变更跟 commit
2. 动文件前 mkdir .collab/locks/<文件域>.lock 申请锁（失败=被占，换域做），用完 rm 释放
3. 任务开始/结束 → 追加 .collab/state.md（会话名前缀 + 真实时间戳）
4. 改动影响其他模块时 → ⚠️ 前缀标注
5. 完成任务 → board.md 移到 ✅，标注 commit hash
6. 开工前 git status 必须干净；push 前 git pull --rebase；一个会话一个端口
```

### 4. 日常命令

| 你说 | Hermes 做 |
|------|----------|
| "同步项目" | 读 state + board + sessions + locks → 汇报状态 |
| "加需求到 {项目}：XXX" | 追加 board ⏳ + 写日志 |
| "更新 wiki" | 读日志 → 写 wiki → 移动完成项 |
| "更新 CLAUDE.md" | 读日志 → 追加式修改 CLAUDE.md |
| "多个 agent 一起开发" | 检查 sessions/locks → 提示按文件域划分会话 |

## 文件结构

```
{项目根目录}/.collab/
├── state.md      ← 运行日志（双方追加，永远插当天区块末尾）
├── board.md      ← 任务公告栏（Hermes 发布，Claude Code 认领）
├── locks/        ← 文件域锁（目录即锁，mkdir 原子）
│   └── <domain>.lock/owner
└── sessions/     ← 会话注册表
    └── <session>.md
```

## 文件格式

### board.md（任务公告栏）

```markdown
# 任务公告栏

> Hermes 分配 → Claude Code 认领 → 做完移 ✅
> 状态：⏳ 待认领 | 🔄 进行中 | ✅ 完成 | ❌ 放弃

## 🔄 进行中

- REQ-004 前端登录页面 (agent-a, 15:45)

## ⏳ 待认领

- REQ-005 文章搜索功能
- REQ-006 关于页 GitHub 贡献图

## ✅ 最近完成

- REQ-003 登录 API (agent-a, a1b2c3d, 7/28)
```

### state.md（运行日志）

```markdown
## 2026-08-01

[14:30] Hermes — 新增需求 REQ-005（文章搜索）
[15:00] agent-a — 完成 REQ-003（登录 API），commit a1b2c3d
[15:45] agent-a — 🔄 开始 REQ-004（前端登录页），涉及 /login/page.tsx
[16:20] agent-a — ⚠️ 修改了 API 返回格式：{code,data,msg} → {status,payload}
[16:45] agent-a — 完成 REQ-004，commit e5f6g7h
```

**时间戳强制规则**：写日志前先 `date "+%m-%d %H:%M"` 拿真实时间，禁止 `[--:--]` 占位符；插到当天 `## YYYY-MM-DD` 区块末尾。

## 文件域定义（锁的粒度，按项目结构自定义）

文件域 = 把项目文件按目录/模块分组，作为锁的粒度。按目标项目实际结构划分：

- 按目录或模块划分（如 `src/components/`、`src/lib/`、某个业务模块）
- 粒度适中：太细（单文件）锁满天飞；太粗（整个仓库）退化为串行
- **全局共享文件单独成域**（如 globals.css、根布局、配置文件）——它们是多任务冲突热点
- 协议文件（`.collab/` 自身）单独成域，供协议维护用

示例划分（前端项目常见形态，仅供参考，务必按实际项目调整）：

| 域 | 覆盖文件 | 冲突风险 |
|----|---------|:------:|
| `styles` | 全局样式、主题配置 | ⚠️ 高 |
| `layout` | 根布局、外壳组件 | ⚠️ 高 |
| `content-data` | 数据层、内容文件 | 中 |
| `components` | 组件目录 | 中 |
| `pages` | 页面目录 | 低 |
| `admin` | 后台 + API | 低 |
| `infra` | 构建/部署/Agent 配置 | 低 |
| `collab` | .collab/ 自身 | — |

全仓级任务（lint 修复、依赖升级、全局重构）没有安全域，应单独排在一个会话空闲时做，不与任何任务并行。

## 设计原则

- **文件系统即协议层** — 不需要 API、不需要数据库、不需要后台进程
- **人类可读** — Markdown 纯文本，可 cat、可 grep、可手动编辑
- **Agent 无关** — 不绑定 Hermes、不绑定 Claude Code
- **用户触发** — 不追求自动，你一句话最可靠
- **简洁优先** — 需求一句话，不给代码
- **机制强制优先于计划约定** — 锁（mkdir 原子）> 归属标注（LLM 不保证遵守）
- **协议通用，细节外置** — skill 不写死项目路径/任务编号，文件域按项目初始化

## 许可

MIT
