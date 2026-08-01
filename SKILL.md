---
name: know-each-other
description: 多 Agent 协作协议 v2.0——通过共享文件（.collab/）实现 Agent 间的任务分配、状态同步、并发会话隔离（文件域锁 + 会话注册 + 认领原子化）。平台无关，任何项目可用。用户一句话触发。
triggers:
  - 同步项目|handoff|项目状态
  - 加需求|新增需求|添加任务
  - 更新 wiki|同步 wiki
  - 更新 CLAUDE.md|维护 CLAUDE
  - 两个 claude|双会话|并发|竞争
---

# Know Each Other — 多 Agent 协作协议 v2.0

让不同 AI Agent 通过共享文件互相感知对方在做什么。不依赖中心调度服务，不依赖特定 Agent 平台。

## 核心原则

> Agent 之间不需要 API。文件系统就是协议层。

Hermes 分配任务、维护需求、记录日志。Claude Code 认领任务、执行开发、汇报进度。两个 Agent 通过 `.collab/` 目录下的文件交接。

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│ Hermes Agent │      │   .collab/    │      │ Claude Code  │
│  分配任务 ────┼─────→│   board.md   │←─────┼── 认领任务   │
│  写日志     ──┼─────→│   state.md   │←─────┼── 汇报进度   │
└─────────────┘      └──────────────┘      └─────────────┘
```

## v2.0 核心变更（并发支持）

v1.x 假设同一时刻只有一个 Agent 活跃。v2.0 支持**多个 Agent 会话并行开发**，新增三个强制机制：

1. **文件域锁**（`locks/`）— 用 `mkdir` 原子性实现互斥，动文件前必须申请对应域的锁
2. **会话注册表**（`sessions/`）— 会话启动/结束必须注册，其他会话可看到谁活着、占着什么
3. **认领原子化** — board.md 变更必须跟一次 git commit，防止互相覆盖

> 为什么不用"任务归属标注"防竞争：标注只是计划，LLM 会话不会真遵守计划。锁是机制强制，mkdir 失败直接挡在门外。

### v2.0 升级背景（2026-08-01 实际触发案例）

用户同时开两个 Agent 会话并行开发时发现竞争风险，升级 v2.0。当时观察到的四类竞争，任何多 Agent 共享一个仓库时都会遇到：

- **board 认领竞态**：多个任务被同一时间戳全部标记 🔄，无会话归属——无法判断哪个会话在干什么
- **文件冲突热点**：两个任务同时改同一个共享文件（如全局样式、根布局），谁后保存谁覆盖谁；全仓级任务（如 lint 清理）与任何并行任务都可能撞
- **git 提交互相卷入**：两个会话各自 commit/push，第二个 push 被拒（non-fast-forward），或一个 commit 把另一个的未提交改动卷进去
- **.collab 无锁**：两会话同时写 state.md / board.md，Markdown 无锁，后者覆盖前者且不可见

**教训**：给任务标"归属"只是计划，LLM 会话不保证遵守——会话崩了、任务边界模糊、没读 board 都会让计划失效。所以 v2.0 用**机制强制**（mkdir 锁）替代计划约定。

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

## 文件域定义（锁的粒度，按项目结构自定义）

文件域 = 把项目文件按目录/模块分组，作为锁的粒度。**按目标项目实际结构划分**，原则：

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

## 用户命令 → Hermes 行为

| 用户说 | Hermes 自动做 |
|--------|-------------|
| "同步项目" / "handoff" | ① 确认项目目录 → ② 读 state.md 最近 50 行 → ③ 读 board.md → ④ 读 sessions/ + locks/ 看并发状态 → ⑤ 汇报：谁做了什么、什么在进行、有无 ⚠️、有无锁冲突、下一步建议 |
| "加需求到 {项目}：XXX" | ① 读 board.md → ② 追加一行到 ⏳ 区 → ③ 追加 state.md 日志 `[时间] Hermes — 新增需求 XXX` |
| "更新 wiki" | ① 读 state.md 了解最近改动 → ② 更新 wiki → ③ board.md 里完成项移到 ✅ 区 → ④ 追加 state.md |
| "更新 CLAUDE.md" | ① 读 state.md（最近的决策/坑）→ ② 读现有 CLAUDE.md → ③ 追加式修改 |
| "多个 agent 一起开发" / "并发" | ① 检查 .collab/sessions/ 是否有活会话 → ② 检查 locks/ 冲突 → ③ 提示按文件域划分会话 → ④ 必要时登记任务到 board |

## state.md 格式

```markdown
## YYYY-MM-DD

[HH:MM] Hermes — 做什么/决策
[HH:MM] agent-a — 做什么，commit xxxxxx
[HH:MM] agent-a — 🔄 开始 {任务}，涉及 {文件}
[HH:MM] agent-a — ⚠️ 修改了 {影响其他模块的改动}
```

- 日志前缀用**会话名**（agent-a / agent-b / Hermes），区分谁写的
- `🔄` 表示任务开始
- `⚠️` 表示改动可能影响其他 Agent / 模块
- 每条一行，简洁可 grep

## ⚠️ 时间戳规则（强制）

**写入 state.md 或 board.md 前，必须执行以下命令获取真实时间，禁止写 `[--:--]` 占位符：**

```bash
date "+%m-%d %H:%M"
```

1. 先跑 `date` 拿到真实月日和时间，再追加日志
2. 追加位置：**永远插到当天 `## YYYY-MM-DD` 区块的末尾**（日期标题后面），不要新建 `(later)`/`(earlier)` 之类重复区块
3. 若跨天（本地凌晨 00:00-06:00），用 `date` 返回的当前日期建新区块，标题格式 `## YYYY-MM-DD`（补零，如 `2026-08-01`）
4. board.md 里标注时间处同样用 `date` 的真实时间，如 `(agent-a, 08-01 22:40)`

> 原因：日志时间用于 handoff 时判断活动顺序，`[--:--]` 和乱序区块会让同步报告失去参考价值。

## board.md 格式

```markdown
# 任务公告栏

> Hermes 分配 → Claude Code 认领 → 做完移 ✅
> 状态：⏳ 待认领 | 🔄 进行中 | ✅ 完成 | ❌ 放弃

## 🔄 进行中

- REQ-004 前端登录页面 (agent-a, 15:45)

## ⏳ 待认领

- REQ-005 文章搜索功能

## ✅ 最近完成

- REQ-003 登录 API (agent-a, a1b2c3d, 7/28)
```

需求条目：一句话摘要。不写详细规格——留给执行 Agent 发挥。

## 多会话工作流（执行 Agent 侧，写进项目 CLAUDE.md 第一条）

### 0. 会话启动 — 注册

```bash
# 1. 读 .collab/sessions/ 看谁活着、占着什么域
# 2. 读 .collab/locks/ 看哪些域被锁
# 3. 写注册文件（会话名由用户指定，如 agent-a / agent-b）
mkdir -p .collab/sessions
echo "- $(date '+%m-%d %H:%M') agent-a, 计划域: components pages" > .collab/sessions/agent-a.md
```

### 1. 认领任务（原子化）

读 board.md → ⏳ 改 🔄 注明会话名 → **board 变更跟一次 commit**：
```bash
git add .collab/board.md
git commit -m "board: 认领 REQ-016 (agent-a)"
```

### 2. 动文件前申请锁（强制）

```bash
mkdir .collab/locks/<domain>.lock        # 成功 = 拿到锁
echo "agent-a $(date '+%m-%d %H:%M')" > .collab/locks/<domain>.lock/owner
# mkdir 失败 = 锁被占 → 换别的域做，或等对方释放
```

释放：`rm -rf .collab/locks/<domain>.lock`

**只碰自己锁的域。没锁的域不碰。锁用完立即释放，会话结束必须清空自己所有锁。**

### 3. 提交纪律（强制）

1. 开工前 `git status` 必须干净 — 有别人未提交的改动先停下问用户
2. 动文件前查 `.collab/locks/` — 被锁的域不碰
3. **push 前必 `git pull --rebase`** — 避免 push 被拒 / 提交互相卷入
4. **一个会话一个端口/环境** — 并行跑 dev server 时用不同端口（如 3000 / 3001），或约定只一个跑 dev、另一个用 build 验证
5. 别同时跑构建（如 `npm run build`）— 构建目录（.next / dist / target）会被踩
6. 会话结束：释放锁 + 删注册 + 带会话名写 state

## Claude Code 的 CLAUDE.md 规则（v2.0 版）

把以下内容放在项目 CLAUDE.md 的第一条（文件域清单按该项目结构替换）：

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

## 初始化

用户说"初始化 know-each-other"时：

1. 确认项目根目录
2. 按项目结构定义文件域清单
3. 创建 `.collab/` 目录 + `state.md` + `board.md` + `locks/` + `sessions/`（locks/sessions 各放 .gitkeep 占位）
4. 追加 state.md 首条日志
5. 提示用户把 CLAUDE.md 规则复制到项目

## ⚠️ 分发规则（强制）

**协议文档更新后必须推送到独立分发仓库**（默认 `https://github.com/qingcheng66/know-each-other.git`，main 分支），保持三份同步：

| 文件 | 内容 |
|------|------|
| README.md | 协议说明 + 思考历程 + 使用方式 |
| SKILL.md | 本 skill 完整版（含 v2.0 并发支持） |
| claude-code.md | Claude Code 的 CLAUDE.md 规则 |

项目仓库只放 `.collab/` 实例，不放协议本体。改 SKILL.md 后同步动作：

```bash
git clone https://github.com/qingcheng66/know-each-other.git /tmp/ke-sync
# 覆盖 README.md / SKILL.md / claude-code.md 三份
cd /tmp/ke-sync && git add . && git commit -m "v2.x.y: ..." && git push origin main
```

> 用户 2026-08-01 明确要求：以后 skill 更新都要推送到专门仓库，不要只留在项目里。

## 设计决策

- **不要自动每轮触发** — Agent 无法被强制。用户一句话触发最可靠
- **不要详细需求规格** — 省 token，发挥执行 Agent 编码能力
- **不要外部守护进程** — 文件系统就是全部基础设施
- **不要数据库** — Markdown 文件可读可编辑可 grep
- **不绑定 Agent 平台** — `.collab/` 目录与 Hermes/Claude Code 解耦
- **锁用 mkdir 不用文件** — mkdir 原子、失败即冲突，比 touch 文件更可靠（touch 会静默覆盖）
- **并发用锁 + 会话注册，不用任务归属** — 归属是计划，锁是机制
- **协议文档通用化，项目细节外置** — skill 不写死具体项目的文件路径/任务编号，文件域清单在初始化时按项目定义（用户 2026-08-01 要求）

## 已知坑位（v2.0）

- **锁可能残留**：会话异常退出（断电/强制 kill）会留下孤儿锁。处理：读 owner 文件看时间，超 2 小时且会话注册文件已删 = 孤儿，手动 `rm -rf`
- **board 认领竞态**：两个会话同时读 board 再写，可能双认领。处理：认领后立即 commit，第二个会话 commit 前 `git pull --rebase` 会看到冲突
- **locks/ sessions/ 空目录进不了 git**：各放一个 `.gitkeep`（或 README.md）占位
- **执行 Agent 默认会自己 commit**：v2.0 要求它 commit 前看 `git status` 和 `git log --oneline -3`，避免把别人改动卷进自己的提交
