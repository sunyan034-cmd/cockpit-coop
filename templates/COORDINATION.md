---
name: cockpit-coordination
status: active
---

# 双 AI 协作协议

> 通用协议，跨项目复用。来源：cockpit-coop skill。
> 核心：**状态写文件，tmux 只 ping。**

## 文件分工（写错地方 = 协作失效）

| 文件 | 写什么 | 不要写什么 |
|---|---|---|
| `SPEC.md` | API 契约 / 数据模型 / 长期决策 | 流水 / 当下进度 |
| `NOW.md` | 当前 active_owner / lock_scope / phase（覆盖式） | 历史 / 契约 |
| `LOG.md` | 已发生事实（commit hash / 验证命令）（append-only） | 契约 / 当下任务 |

**cockpit/ 只放上面四份协作文档。**patch / draft 数据 / 临时产物 → 放 `drafts/` 或仓库根，cockpit/ 里只指针引用。

## tmux ping 铁律

**一行，<80 字符，必带 `read cockpit/NOW.md`，commit + push 之后才发。**

模板：

```text
[Codex] done e44b9c3; next=Claude; read cockpit/NOW.md
[Claude] need=schema review; read cockpit/NOW.md
[Codex] interrupt? read cockpit/NOW.md
```

发命令：

```bash
tmux send-keys -t <session>:0.1 '[Claude] done abc; next=Codex; read cockpit/NOW.md' Enter
```

**绝对不要**：

```bash
# ❌ 反模式：长消息塞 send-keys
tmux send-keys -t leyan:0.1 "搞完了，前端部署了，后端 draft 在 stash 里，契约是 GET /api/x 返回 {y:z}，TTL 10 分钟，你看下..." Enter
```

## 交接动作序列（缺一步 = 没交接）

每次完成一段工作：

1. 写代码 / 跑测试
2. `git diff --cached` 看清楚（不带对方脏文件）
3. `git commit + push`
4. **更新 `NOW.md`**：active_owner 切对方 / phase / lock_scope
5. **append `LOG.md`** 一条事实（commit hash + 验证命令）
6. 改了契约 → **改 `SPEC.md`** + LOG 注一笔"已改 SPEC §X.Y"
7. tmux 发短 ping

## SPEC.md 改动检查点

- `SPEC.md` 是长期契约，改前必须先让对方 ack。
- 先在 `NOW.md` 加 `## SPEC change proposal`，写要改什么 + 为什么；ping：`[X] propose SPEC §Y change; read cockpit/NOW.md`。
- 对方在 `NOW.md` 写 ack / push back 后，才能改 `SPEC.md`。
- 紧急改字段可先改，但 commit message 标注 `[X] SPEC emergency: ...`，立刻 ping，事后补流程。

## Lock scope（互不重叠才能并行）

NOW.md 必须有这张表：

| Owner | Scope | 状态 |
|---|---|---|
| Claude | `web/` 前端 | done @ <hash> / in_progress |
| Codex | `services/api/` 后端 | next / in_progress |

要并行改 → 拆成不重叠的 scope。
要把活交给对方 → owner 切对方。

## Worktree 纪律

- 同时只一个 active owner，另一个默认只读。
- 不碰对方未提交脏文件。
- `.playwright-cli/`、`.codex-cli/` 之类工具产物**必须**加 .gitignore（首次发现立刻加）。
- 并行多了改用两个 worktree：`<repo>-claude` / `<repo>-codex`。

## Commit 边界

- 一阶段一 commit。
- commit message 头注 owner + 范围：`[Claude] M3 brief 截断按句号`。
- 提交前 `git diff --cached` 看清楚。
- 跑完相关测试再 commit；测试命令写进 LOG.md。
- **push 之后再 ping**，不 ping 未推状态。

## 冲突处理

- 不打断对方长任务（看对方 pane 在跑命令就别戳输入框）。
- 必须打断只发：`[Codex] interrupt? read cockpit/NOW.md`。
- 状态不一致 → 以 **GitHub main + NOW.md** 为准；LOG.md 只用来追溯。

**30 分钟升级阈值**：
- interrupt ping 后 30 分钟，对方 pane 仍 idle、`NOW.md` 没动 → 升级到大哥。
- 主对话报告：`$项目 协作卡住 30min，已 ping 对方无应，等大哥介入`。
- 不自己解锁 / 不替对方决策 / 不无限等。

## 大哥验收检查点

- `NOW.md` 显示「等大哥验收」时，默认不主动开新工作。
- 必须私聊大哥报状态：`$项目 当前 N 项目等大哥验收：xxx，要不要继续？`
- 大哥裁决后，在 `NOW.md` 加 `## 大哥裁决 YYYY-MM-DD HH:MM`，写明决定再开工。
- 大哥不在线 / 未回 → 继续 idle，不替大哥决定。

## /compact 处理

各自 compact 不影响对方。compact 后回来：

```bash
cat cockpit/NOW.md
tail -50 cockpit/LOG.md
```

即可恢复状态。
