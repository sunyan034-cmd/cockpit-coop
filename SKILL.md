---
name: cockpit-coop
description: Use when two AI agents (Claude + Codex / Claude + Gemini / Claude + Claude) work in the same tmux session on the same repo, or when entering a project that already has cockpit/COORDINATION.md. Triggers include "和 codex 一起做"、"开两个 pane 协作"、"双 AI 分工"、"让另一个 AI 帮忙"、"两个 agent 同时干"。
---

# cockpit-coop — 双 AI 在同一 tmux 协作协议

## Overview

两个 AI 在同一个 tmux session 里改同一份代码会踩六个高频坑（4 + 2 实战补）。本 skill 用 cockpit/ 目录承载状态，让 tmux 只做敲门通知。

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

## 六条铁律（1-4 来自 baseline 测试，5-6 来自 leyan-image M7 实战）

### 铁律 1：tmux 只发一行短 ping，长说明全部写文件

**Baseline 失败 #1**：subagent 在 tmux 一次性塞 6 个事实 + 完整 API 契约 + 注意事项。结果：对方输入框被塞满 / queued message 过期 / 信息丢失。

**正确做法**：
- 长说明（契约 / 流程 / 完整状态）→ 写 cockpit/ 文件
- tmux ping 只发**一行 < 80 字符**，**必带 `read cockpit/NOW.md`**

**Ping 模板（强制格式，必含动词）**：

```
# 状态报告档（自报进度）
[Codex] done <hash>; next=Claude; read cockpit/NOW.md
[Claude] need=schema review; read cockpit/NOW.md
[Codex] blocked=<原因> @<hash> read cockpit/NOW.md

# 指令动词档（必须含动词，别只甩 read NOW.md）
[Codex] 开干 M7a/b @<hash> read NOW.md          # 大哥已拍开干，让对方立刻动手
[Codex] 暂停 M7c 等审查 @<hash> read NOW.md     # 让对方停手等 review
[Codex] M7a done 求 review @<hash> read NOW.md  # 自己 done 让对方接

# 紧急
[Codex] interrupt? read cockpit/NOW.md
```

⚠️ **禁用模糊 ping**：`read NOW.md`、`方向变了`、`看一下`、`同步一下` —— 对方收到无动词的 ping 会回 NOW.md 找指令，NOW.md 滞后于人类口头指令时对方就保守地停下来 idle。**ping 必须含动作，不能只是文档指针**。

发命令（Codex CLI v0.125+ TUI 必须用 paste-buffer + C-m + **capture-pane 验证**）：

```bash
# 完整 SOP：set-buffer → paste-buffer → sleep 1 → C-m → sleep 1.5 → 验证
P=<session>:0.1   # 或按 path 找：P=$(tmux list-panes -a -F "#{pane_id} #{pane_current_command} #{pane_current_path}" | awk '$2=="node" && $3=="<repo path>" {print $1; exit}')
tmux set-buffer "[Claude] 开干 M7a @abc123 read NOW.md"
tmux paste-buffer -t $P
sleep 1                                              # 关键 1：给 paste 异步落地时间
tmux send-keys -t $P C-m
sleep 1.5                                            # 关键 2：给 Codex 处理时间
tmux capture-pane -p -t $P | tail -8 | grep -q "Working\|esc to interrupt" \
  && echo "✅ Codex Working" \
  || { echo "⚠️ ping 未提交，救场连发"; \
       for K in C-m Enter C-j C-m; do tmux send-keys -t $P $K; sleep 0.3; done; \
       sleep 1.5; tmux capture-pane -p -t $P | tail -5; }
```

🔥 **铁律：没看到 capture-pane 输出含 "Working" 或 "esc to interrupt" 字样，绝不报"已 ping"** —— 盲报成功 = 让人类目测背锅 = 一直会被骂同样的问题。

⚠️ **三个常踩的坑**：
1. **不要用** `tmux send-keys -t $P '...' Enter` —— Codex CLI TUI 用 raw mode 监听具体 keycode，Enter 别名映射 LF (`\n`)，Codex 只认 CR (`\r`)，`C-m` 才是真正的 `\r`。普通 zsh/bash shell 不受影响（readline 接受 LF）。
2. **paste-buffer 是异步的** —— 紧跟着 send-keys C-m 时 paste 还没落地，C-m 就被忽略。**sleep 1 是必需的不是优化**。
3. **C-m 之后也要 sleep + capture 验证** —— Codex 处理 keystroke 有延迟，立刻 capture 可能还看到老画面，至少 sleep 1.5 再 capture。

**清空对方输入框**用 `C-u`（行删除，不会中断 Codex）；**不要**用 `C-c`（会被 TUI 当 SIGINT 中断对方任务）。

**保持 ping <80 字符 单行** —— 长 ping 即使提交了也容易被 Codex TUI 当成行内换行处理；长说明全部塞 `cockpit/NOW.md`。

> 反模式参考：见底部「反模式速查」表

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

> 反模式参考：见底部「反模式速查」表

### 铁律 3：push 之后才 ping，不 ping 未推状态

**Baseline 失败 #5（场景 B）**：subagent 接活时开工前就 ping"Claude 上线"。开工前没东西交付，ping 是噪音；干完 push 之前 ping，对方 git pull 拉不到。

**正确做法**：
- 开工**不**ping（NOW.md 切 owner 后对方自己看）
- 干完 + push 后**才** ping
- 必须打断对方时只发：`[Codex] interrupt? read cockpit/NOW.md`

> 反模式参考：见底部「反模式速查」表

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

> 反模式参考：见底部「反模式速查」表

### 铁律 5：owner=对方时主动轮询，不傻 idle

**实战教训（leyan-image M7 启动 2026-04-28）**：Claude ping Codex 后 NOW.md 写 `owner=Codex`，就转 idle 等 `done @ <hash>`。但 Codex 实际停下来等更明确指令（NOW.md 滞后于人类口头指令），Claude 不知情，结果人类不爽："还非得我盯着吗"。

**正确做法**：

owner=对方期间，Claude **每 5-10 分钟主动 check**：

```bash
git log --oneline -3                                 # 对方有无新 commit
tmux display-message -p -t <session>:0.1 \
  '#{pane_current_command}'                          # 对方进程是否还在跑
tmux capture-pane -t <session>:0.1 -p | tail -10     # 对方实时输出（alternate screen TUI 可能为空）
```

**发现以下任一情况立刻处理**，不等人类提醒：

- 对方进程已退出 (`pane_current_command=zsh/bash` 而不是 `node`/`python`) → 大概率 stuck 或 CLI 已退，重 ping 或问人类
- 对方 Working 时间 >10 分钟无新 commit → 看 capture 输出找原因（卡在某个 build/test？需要 ack？）
- 对方在 prompt 等指令（不是 Working 状态）→ ping 内容大概率不够明确，重发含动词的 ping
- 对方 done 但没主动 ping 你 → 主动接活

**真正的 idle = 主动盯，不是傻等**。subagent 默认行为是"owner=对方就只读"——这条铁律是对默认行为的覆盖。

> 反模式参考：见底部「反模式速查」表

### 铁律 6：人类口头指令必须先落 NOW.md 再 ping

**实战教训（leyan-image M7 启动 2026-04-28）**：人类在对话里说"OK 你现在开始做"，Claude 没立刻把"开干"翻译进 NOW.md，只 ping Codex `M7 大方向变 read NOW.md`。Codex 读完 NOW.md 看到旧版本仍写"等人类拍开干"，就保守地停下来。结果人类以为 Codex 在干，实际 Codex idle。

**正确做法**：人类在对话里给口头指令（开干/暂停/换方案/重做/合并/拆分）→ Claude **立刻三步走**：

1. **先改 NOW.md**：把指令书面化（例 "人类已拍 M7 GO @<时间>"），更新 `active_owner` / `phase` / Lock scope 表
2. **再 commit**（让对方 `git pull` 或 `git log` 拿得到新 NOW.md）
3. **再 ping**：内容必须含动词（参考铁律 1 指令动词档）

⚠️ **反模式**：直接 ping 对方转告人类的话（如 `[Codex] 大哥说开干 read NOW.md`）——对方严格按 NOW.md 行动，转告的话没落文件就不会被对方当真。**Claude 是人类指令的"翻译官"**，必须把口头指令书面化进 NOW.md，再用 ping 触达通知。

> 反模式参考：见底部「反模式速查」表

## 检查点速查

### CP-1: 改 SPEC.md 前先 ack
**对应铁律**：铁律 4（lock_scope 显式）

**触发**：准备修改 API 契约、数据模型、长期决策或 SPEC.md 任意稳定约定。
**动作**：1. 在 NOW.md 写 proposed_spec_change + owner；2. tmux 短 ping 要对方 ack；3. 对方 ack 后再改 SPEC.md；4. LOG 记录已改 SPEC §X。
**话术 / 模板**：`[Codex] need=SPEC ack §2.1; read cockpit/NOW.md`
**fallback**：对方无响应且非紧急，不改契约；紧急只先写草案到项目临时目录并标明未生效。

### CP-2: NOW.md 写等大哥验收时先 confirm
**对应铁律**：铁律 3（push 后才 ping）

**触发**：NOW.md 的 phase / next 写明等待用户验收、确认、拍板或暂停。
**动作**：1. 停止主动开新任务；2. 汇总当前状态和验证结果；3. 向大哥确认是否继续；4. 得到明确指令后再切 owner / phase。
**话术 / 模板**：`[Claude] waiting=user confirm; read cockpit/NOW.md`
**fallback**：若只是修阻塞性小错，可先提出最小修复方案；没有确认前不扩大范围。

### CP-3: 冲突 30 分钟无应升级
**对应铁律**：铁律 1（tmux 只发短 ping）

**触发**：lock_scope、owner、契约解释或交接状态冲突，且对方 30 分钟内无响应。
**动作**：1. 不继续写冲突范围；2. NOW.md 标记 blocked + 冲突点；3. LOG 记录时间和已尝试 ping；4. 报告大哥请求裁决。
**话术 / 模板**：`[Codex] blocked=scope conflict; need human; read cockpit/NOW.md`
**fallback**：只允许继续不重叠 scope 的只读分析或验证；禁止凭猜测改对方范围。

### CP-4: lock_scope 同时占用的先到先得
**对应铁律**：铁律 4（lock_scope 显式写进 NOW.md）的并发延伸

**触发**：
- 你打算改 X 文件，发现 NOW.md 上 lock_scope 表里 X 已被对方占
- 或两个 agent 同时 push commit，commit message 都标了同一个 scope

**动作**：
1. 看 NOW.md 里那条 lock_scope 的写入时间（`git log -1 cockpit/NOW.md`）
2. **谁先 commit 进 NOW.md，谁拥有该 scope**；后到方让出，改自己 NOW.md 行 status=`yield to <对方>`
3. 后到方在 LOG.md 加一条 `[X] yield scope <name> to <对方> (写入早 N 分钟)`
4. 后到方挑互不重叠的另一 scope 干，或转 idle 等对方 done @ <hash>

**fallback**：
- 两个 NOW.md commit 时间戳一致 → 字典序靠后的 owner 让
- 双方都已经动手改了文件 → 后到方 stash 自己改动，等对方 done 后再决定 rebase / 重做

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
   - ⚠️ **alternate screen TUI 抓不到**：Codex CLI v0.125+ 用 alternate screen，capture 经常返回大段空白
   - 替代方案：`git log --oneline -5`（看 commit 推进）+ `tmux display-message -p -t <session>:0.1 '#{pane_current_command}'`（看对方进程是 `node`/`python` 还是退回 `zsh`）

判断分支：
- NOW.md owner=自己 → 开干
- NOW.md owner=对方 → **不是只读，按铁律 5 主动轮询**（每 5-10 分钟 check）
- NOW.md 写"等人类验收"→ 报告人类状态，不主动开新工作（CP-2）

## 反模式速查（baseline + 实战观察到的）

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
| **ping 不含指令动词**（只甩 `read NOW.md`） | 对方回 NOW.md 找指令，NOW.md 滞后就保守地停下来 idle | 必须含"开干/暂停/审查"动词（铁律 1） |
| **人类口头指令直接 ping 不落 NOW.md** | 对方严格按 NOW.md 行动，转告的话不当真 | 先改 NOW.md commit，再 ping（铁律 6） |
| **owner=对方就傻 idle 等 ping** | 对方 stuck/退出不知情，人类要盯 | 每 5-10 分钟主动 `git log` + 看 pane 进程（铁律 5） |
| **长 ping (>1 行)** + `paste-buffer + C-m` | C-m 被 TUI 当行内换行不是提交，消息卡输入框 | 短 ping <80 字符单行；卡了用救场招连发回车 |
| **`send-keys ... Enter` 给 Codex CLI** | Enter 映射 LF，Codex 只认 CR，消息进了不提交 | 用 `paste-buffer + C-m`（铁律 1 发命令段） |
| **`capture-pane` 抓 Codex CLI 输出** | alternate screen 经常返回空白，看不到状态 | 用 `git log` + `pane_current_command` 替代 |
| **`C-c` 清对方输入框** | 被 TUI 当 SIGINT 中断对方任务 | 用 `C-u` 行删除（普通字符不被中断） |

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
