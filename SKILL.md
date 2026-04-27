---
name: cockpit-coop
description: Use when two AI agents (Claude + Codex / Claude + Gemini / Claude + Claude) work in the same tmux session on the same repo, or when entering a project that already has cockpit/COORDINATION.md. Triggers include "和 codex 一起做"、"开两个 pane 协作"、"双 AI 分工"、"让另一个 AI 帮忙"、"两个 agent 同时干"。
---

# cockpit-coop — 双 AI 在同一 tmux 协作协议

## Overview

两个 AI 在同一个 tmux session 里改同一份代码会踩四个高频坑。本 skill 用 cockpit/ 目录承载状态，让 tmux 只做敲门通知。

**核心一句话：状态写文件，tmux 只 ping。**

## 经验来源（why this exists）

来自 `~/claudecode/leyanai-station` 项目 2026-04-27 Claude × Codex 实战。
原始踩坑、决策见原项目 `cockpit/LOG.md`。

## 何时使用本 skill

- 用户说要让 Claude 跟另一个 AI（Codex / Gemini / 另一个 Claude）一起做项目
- 用户提到"开两个 tmux pane"、"双 AI"、"分工"、"让 X 帮忙"
- 进入项目根目录发现 `cockpit/COORDINATION.md` → 自动按协议接活
- 用户问"我和另一个 AI 怎么配合不打架"

## 物理布局

```
tmux session = <proj>
├── pane :0.0  → Claude (本 agent)
└── pane :0.1  → 另一个 AI（Codex 等）
```

两个 pane 同一工作目录，看同一份代码。

## cockpit/ 文件分工（baseline 测试发现 agent 会写错地方）

| 文件 | 写什么 | **不要**写什么 |
|---|---|---|
| `SPEC.md` | **API 契约 / 数据模型 / 长期决策**（如 `GET /api/weather` 字段表） | 不要写流水、不要写当下进度 |
| `NOW.md` | **当前 active_owner / lock_scope / phase**（覆盖式） | 不要 append 历史、不要写契约 |
| `LOG.md` | **已发生事实**（commit hash / 验证命令 / 决策点） | **不要写 API 契约**（属于 SPEC）、不要写当下任务（属于 NOW） |
| `COORDINATION.md` | 协作协议本身（拷自本 skill 模板） | 一次写定不再改 |

> **cockpit/ 只放协作文档**（上面四份 .md）。
> patch / draft 数据 / 临时产物 / 截图 → 放别处（如项目根的 `tmp/`、`drafts/` 或 stash），不进 cockpit/。
> cockpit/ 是双方都信任的协作正源，混入数据会让分工边界模糊。

> **Baseline 失败 #3**：subagent 自然会把"前端调后端的契约"写进 LOG.md。**这是错位**。契约属于 SPEC.md，LOG 只记"已改 SPEC §X.Y 字段"。
>
> **GREEN 验证发现的次级 loophole**：subagent 加载 skill 后，会自然把 draft patch 放进 `cockpit/weather-cache-draft.patch`。它意识到不该污染 SPEC/NOW/LOG，但仍把临时数据塞进 cockpit/。**正确做法**：patch 放 `drafts/` 或仓库根，cockpit/ 里只在 LOG 注一笔"draft @ drafts/xxx.patch"。

## 四条铁律（每条对应一个 baseline 失败模式）

### 铁律 1：tmux 只发一行短 ping，长说明全部写文件

**Baseline 失败 #1**：subagent 在 tmux 一次性塞 6 个事实 + 完整 API 契约 + 注意事项。结果：对方输入框被塞满 / queued message 过期 / 信息丢失。

**正确做法**：
- 长说明（契约 / 流程 / 完整状态）→ 写 cockpit/ 文件
- tmux ping 只发**一行 < 80 字符**，**必带 `read cockpit/NOW.md`**

**Ping 模板（强制格式）**：
```
[Codex] done <hash>; next=Claude; read cockpit/NOW.md
[Claude] need=schema review; read cockpit/NOW.md
[Codex] interrupt? read cockpit/NOW.md
```

发命令：
```bash
tmux send-keys -t <session>:0.1 '[Claude] done abc123; next=Codex; read cockpit/NOW.md' Enter
```

**反模式**：
```bash
# ❌ 永远不要这么发
tmux send-keys -t leyan:0.1 "天气源已切 open-meteo，前端部署完了。后端 weather.py 缓存我写了个 draft，stash 起来导出到 cockpit/weather-cache-draft.patch，麻烦你审完接手。前端调用契约见 cockpit/LOG.md 最新一条..." Enter
```

### 铁律 2：每次交接必须更新 NOW.md（不只是 LOG.md）

**Baseline 失败 #2**：subagent 写了 LOG.md 流水但没动 NOW.md。下次对方进来要翻 LOG 才知道当前 owner 是谁。

**正确做法**：完成一段工作的标准动作序列：

1. 写代码 / 跑测试
2. `git diff --cached` 看清楚（不带对方脏文件）
3. `git commit + push`
4. **更新 `cockpit/NOW.md`**：active_owner 切到对方 / phase 改成下一阶段 / lock_scope 表更新
5. **append `cockpit/LOG.md`** 一条事实记录（含 commit hash + 验证命令）
6. 如果改了 API 契约 / 字段 / 长期决策 → **改 `cockpit/SPEC.md`** + 在 LOG.md 注一笔"已改 SPEC §X.Y"
7. tmux 发短 ping

**漏第 4 步 = 没交接。** 即使代码已 push、LOG 已记，对方进来照样不知道 owner 该自己接了。

### 铁律 3：push 之后才 ping，不 ping 未推状态

**Baseline 失败 #5（场景 B）**：subagent 接活时开工前就 ping"Claude 上线"。开工前没东西交付，ping 是噪音；干完 push 之前 ping，对方 git pull 拉不到。

**正确做法**：
- 开工**不**ping（NOW.md 切 owner 后对方自己看）
- 干完 + push 后**才** ping
- 必须打断对方时只发：`[Codex] interrupt? read cockpit/NOW.md`

### 铁律 4：lock_scope 显式写进 NOW.md

**Baseline 失败 #4**：subagent 行为上分边界对（前端/后端分开 commit），但没在 NOW.md 写出"哪个文件归谁"。后果：对方不知道你的 draft 还要不要继续，只能猜。

**正确做法**：每次更新 NOW.md 时填这张表：

```markdown
## Lock scope

| Owner | Scope | 状态 |
|---|---|---|
| Claude | web/index.html 前端 | done @ abc123 |
| Codex | services/api/weather.py 后端 | next（接 Claude draft，stash@{0}）|
```

要并行改 → 拆 lock_scope 互不重叠。
要把活交给对方 → 显式写 owner=对方。

## 检查点速查

### CP-1: 改 SPEC.md 前先 ack

**触发**：准备修改 API 契约、数据模型、长期决策或 SPEC.md 任意稳定约定。
**动作**：1. 在 NOW.md 写 proposed_spec_change + owner；2. tmux 短 ping 要对方 ack；3. 对方 ack 后再改 SPEC.md；4. LOG 记录已改 SPEC §X。
**话术 / 模板**：`[Codex] need=SPEC ack §2.1; read cockpit/NOW.md`
**fallback**：对方无响应且非紧急，不改契约；紧急只先写草案到项目临时目录并标明未生效。

### CP-2: NOW.md 写等大哥验收时先 confirm

**触发**：NOW.md 的 phase / next 写明等待用户验收、确认、拍板或暂停。
**动作**：1. 停止主动开新任务；2. 汇总当前状态和验证结果；3. 向大哥确认是否继续；4. 得到明确指令后再切 owner / phase。
**话术 / 模板**：`[Claude] waiting=user confirm; read cockpit/NOW.md`
**fallback**：若只是修阻塞性小错，可先提出最小修复方案；没有确认前不扩大范围。

### CP-3: 冲突 30 分钟无应升级

**触发**：lock_scope、owner、契约解释或交接状态冲突，且对方 30 分钟内无响应。
**动作**：1. 不继续写冲突范围；2. NOW.md 标记 blocked + 冲突点；3. LOG 记录时间和已尝试 ping；4. 报告大哥请求裁决。
**话术 / 模板**：`[Codex] blocked=scope conflict; need human; read cockpit/NOW.md`
**fallback**：只允许继续不重叠 scope 的只读分析或验证；禁止凭猜测改对方范围。

## 工作流程

### A. 从零搭一套（首次双 AI 协作）

**推荐：用自带脚本一键搭骨架**：

```bash
cd /path/to/project   # 必须是 git 仓库
~/.claude/skills/cockpit-coop/bin/cockpit-init [tmux-session-name]
```

脚本自动：
- 创建 `cockpit/` 并拷 4 份模板
- 在项目 `CLAUDE.md` / `AGENTS.md` 顶部插入协议指针（让 Codex 进项目自动看到）
- 加 `.gitignore`（`.playwright-cli/`、`.codex-cli/`、`.codex/sessions/`、`.claude/sessions/`）
- 输出 next steps（编辑 SPEC §1 / 起 tmux / commit 骨架）

**手动版（不想用脚本）**：

```bash
mkdir -p cockpit
cp ~/.claude/skills/cockpit-coop/templates/*.md cockpit/
# 然后手动：填 SPEC §1、改 NOW.md、加 .gitignore、起 tmux
```

### B. 进入已有 cockpit/ 项目（接活）

按这个顺序读，**不要乱序**：

1. `cat cockpit/NOW.md`（看当前 owner / phase / lock_scope）
2. `tail -50 cockpit/LOG.md`（看最近发生了什么）
3. `cat cockpit/COORDINATION.md`（首次进 / 忘了规则时）
4. `cat cockpit/SPEC.md`（改契约相关代码前）
5. `tmux capture-pane -t <session>:0.1 -p | tail -50`（看对方实时状态）

判断分支：
- NOW.md owner=自己 → 开干
- NOW.md owner=对方 → 默认只读，等 ping
- NOW.md 写"等大哥验收"→ 报告大哥状态，不主动开新工作

## 反模式速查（baseline 直接观察到的）

| 错误 | 后果 | 正确做法 |
|---|---|---|
| tmux 发多句长消息 | 卡输入框 / queued 过期 | 一行 ping + 文档指针 |
| 写 LOG.md 不更新 NOW.md | 对方不知道 owner 切了 | NOW + LOG 同时改 |
| API 契约写进 LOG.md | 流水里捞契约找不到 | 契约写 SPEC.md |
| ping 不带 `read cockpit/NOW.md` | 对方靠 ping 正文找信息 | 强制带文档指针 |
| 开工前 ping | 没东西交付，噪音 | 干完 push 后才 ping |
| 没 push 就 ping | 对方 pull 不到 | push 后再 ping |
| lock_scope 不显式 | 对方猜 draft 要不要保留 | NOW.md 写表格 |
| commit 时不 git diff --cached | 带走对方脏文件 | 检查再 commit |

## 模板文件

`~/.claude/skills/cockpit-coop/templates/` 自带 4 份：
- `COORDINATION.md` — 协作协议（拷贝即用，无需改）
- `SPEC.md` — 稳定契约骨架
- `NOW.md` — 当前接力骨架
- `LOG.md` — 事实流水骨架

初始化：
```bash
mkdir -p cockpit && cp ~/.claude/skills/cockpit-coop/templates/*.md cockpit/
```
