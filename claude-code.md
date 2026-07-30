# Claude Code 用 — know-each-other 协作协议

把以下内容放在项目 CLAUDE.md 的第一条。

---

## 🔴 协作协议（know-each-other）

1. **每轮开始前**读 `.collab/board.md`，认领 ⏳ 任务 → 标记 🔄
2. **任务开始/结束时** → 追加 `.collab/state.md`
3. **改动影响其他模块时** → ⚠️ 前缀标注
4. **完成任务后** → `board.md` 移到 ✅ 区，标注 commit hash

### state.md 追加格式

```
[HH:MM] Claude Code — 🔄 开始 {任务}，涉及 {文件}
[HH:MM] Claude Code — ⚠️ {影响其他模块的改动说明}
[HH:MM] Claude Code — 完成 {任务}，commit {hash}
```

### board.md 认领格式

认领时把任务从 ⏳ 移到 🔄，标注时间和 Agent 名：
```
- REQ-004 前端登录页面 (Claude Code, 14:30)
```

完成时移到 ✅：
```
- REQ-004 前端登录页面 (Claude Code, a1b2c3d, 7/28)
```

### 注意

- `🔄` = 任务开始
- `⚠️` = 改动可能影响 Hermes 或其他模块
- 每轮对话的第一个动作：读 board.md
- 需求只给一句话摘要，实现细节你自己决定
