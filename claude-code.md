# Claude Code 用 — know-each-other 协作协议 v2.0

把以下内容放在项目 CLAUDE.md 的第一条。

---

## 🔴 协作协议（know-each-other v2.0）

0. **启动**：读 `.collab/sessions/` + `.collab/locks/` 看谁活着、哪些文件域被锁；写自己的注册文件 `.collab/sessions/<会话名>.md`（会话名由用户指定，如 claude-a / claude-b）
1. **认领**：读 `.collab/board.md` 认领 ⏳ 任务 → 标记 🔄 + 会话名；**board 变更必须跟一次 git commit**（防双会话覆盖）
2. **锁**：动文件前 `mkdir .collab/locks/<文件域>.lock` 申请锁（成功=拿到锁，`mkdir: File exists`=被占→换域做或等释放），写入 owner，用完 `rm -rf` 释放。**只碰自己锁的域**
3. **日志**：任务开始/结束时 → 追加 `.collab/state.md`（**会话名前缀** + 真实时间戳）
4. **⚠️**：改动影响其他模块时 → ⚠️ 前缀标注
5. **完成**：完成任务 → board.md 移到 ✅，标注 commit hash
6. **提交纪律**：开工前 `git status` 必须干净（有别人未提交的改动先停下）；push 前必 `git pull --rebase`；一个会话一个端口（第一个 dev 3000，第二个 `npm run dev -- -p 3001`）；别同时 `npm run build`（.next 会被踩）
7. **结束**：释放自己所有锁 + 删除注册文件 + 带会话名写 state

### 文件域定义

`styles`(globals.css/tailwind) | `layout`(layout.tsx/根组件) | `content-data`(content.ts/content/*.json) | `components`(src/components/) | `pages`(src/app/) | `admin`(后台+API) | `infra`(Dockerfile/CLAUDE.md/配置) | `collab`(.collab/自身)

### 会话启动注册格式

```bash
mkdir -p .collab/sessions
echo "- $(date '+%m-%d %H:%M') claude-a, 计划域: components pages" > .collab/sessions/claude-a.md
```

### state.md 追加格式（先 `date "+%m-%d %H:%M"` 拿真实时间，插当天区块末尾）

```
[HH:MM] claude-a — 🔄 开始 {任务}，涉及 {文件}
[HH:MM] claude-a — ⚠️ {影响其他模块的改动说明}
[HH:MM] claude-a — 完成 {任务}，commit {hash}
```

### board.md 认领格式

认领时把任务从 ⏳ 移到 🔄，标注会话名和时间：
```
- REQ-004 前端登录页面 (claude-a, 14:30)
```

完成时移到 ✅：
```
- REQ-004 前端登录页面 (claude-a, a1b2c3d, 7/28)
```

### 申请/释放锁

```bash
mkdir .collab/locks/components.lock        # 成功 = 拿到锁
echo "claude-a $(date '+%m-%d %H:%M')" > .collab/locks/components.lock/owner
# mkdir 失败 = 锁被占 → 换别的域做，或等对方释放

rm -rf .collab/locks/components.lock        # 用完释放
```

### 注意

- `🔄` = 任务开始
- `⚠️` = 改动可能影响 Hermes 或其他模块
- 每轮对话的第一个动作：读 board.md + sessions/ + locks/
- 需求只给一句话摘要，实现细节你自己决定
- **锁用完立即释放；会话结束必须清空自己所有锁 + 删注册文件**
