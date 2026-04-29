# cockpit-coop

让两个 AI agent（Claude + Codex / Gemini / 另一个 Claude）在同一个 tmux session 里改同一份代码时不打架。

**核心铁律：状态写文件，tmux 只 ping。**

## 这是什么

业界 multi-agent handoff（OpenAI Agents SDK / LangGraph / Microsoft Agent Framework）解决的是"同一个 SDK 内的 agent 切换"。

**cockpit-coop 解决"跨进程跨厂商的 agent 协作"**——比如你在一个 tmux 里同时开 `claude` 和 `codex`，让它俩分工干活：一个前端一个后端。

不依赖任何 SDK，只要两个 agent 都能读写文件 + 看 tmux pane，协议就成立。

可选再起第三 pane 跑 **Hermes / 调度员** 当翻译层：读 `cockpit/`、看 tmux、看 git，把状态翻译给大哥并判断下一步 owner。Hermes **不另开状态目录**，正源仍是 `cockpit/`；默认不动业务代码，必要时只最小更新 `cockpit/NOW.md` / `LOG.md`。详见 [SKILL.md](./SKILL.md) 「Hermes / 调度员」一节。

## 依赖

**必需**：

| 工具 | 用途 | 安装 |
|---|---|---|
| **git** | 协议核心（push 后才 ping）| 系统自带，或 `brew install git` |
| **tmux** | 双 pane 物理布局 | `brew install tmux`（macOS）/ `apt install tmux`（Linux）|
| **bash** | 初始化脚本 `cockpit-init` | 系统自带 |
| **Claude Code CLI** | 加载 skill 的载体 | https://claude.com/claude-code |

**强烈推荐**（选一个跟 Claude 配合的另一个 AI）：

| Agent | 安装 |
|---|---|
| **Codex CLI**（推荐） | `npm install -g @openai/codex` 或参考 https://github.com/openai/codex |
| Gemini CLI | https://github.com/google-gemini/gemini-cli |
| 另一个 Claude Code | 起两个 pane 都跑 claude 也行 |

任何能读写文件 + 在 tmux 里用 `tmux capture-pane` 看到的 AI agent 都能配合本协议。

## 安装

```bash
git clone https://github.com/sunyan034-cmd/cockpit-coop.git ~/.claude/skills/cockpit-coop
```

装完后 Claude Code 会自动识别这个 skill。说"和 codex 一起做"、"双 AI 协作"、"开两个 pane 协作"等触发词，Claude 会自动按协议干活。

## 更新

```bash
cd ~/.claude/skills/cockpit-coop && git pull
```

## 怎么用

### 方式 1：一键搭骨架（推荐）

```bash
cd /path/to/your/project   # 必须是 git 仓库
~/.claude/skills/cockpit-coop/bin/cockpit-init
```

脚本会自动：
- 创建 `cockpit/` 目录 + 拷 4 份模板（COORDINATION/SPEC/NOW/LOG）
- 在项目 `CLAUDE.md` / `AGENTS.md` 顶部插入协议指针（让 Codex 进项目自动看到）
- 加 `.gitignore`（屏蔽 `.playwright-cli/`、`.codex-cli/`、各类 sessions 目录）
- 输出下一步提示

### 方式 2：起 tmux 双 pane

```bash
tmux new -s myproj -d
tmux split-window -h -t myproj:0
tmux attach -t myproj
```

在 `:0.0` 启动 Claude（`claude`），在 `:0.1` 启动 Codex（`codex`）。

第一句让两个 agent 都先读 `cockpit/COORDINATION.md` 了解协议。

需要调度员就再 `tmux split-window -v -t myproj:0` 起第三 pane 跑 Hermes，喊"调度 image / 翻译 Codex / 巡检项目"。Hermes 按 `pane_current_path` + 项目名定位，不抢 Claude/Codex 的活。

### 方式 3：直接对 Claude 说

在已经有 `cockpit/COORDINATION.md` 的项目里，直接对 Claude 说：

> "你和 Codex 在这个项目协作，看下当前状态，决定下一步。"

Claude 会按 skill 协议读 NOW.md / LOG.md 判断接力状态。

## 协议核心

### 4 条铁律
1. **tmux 只发一行短 ping**，长说明全部写文件
2. **每次交接必须更新 NOW.md**（不只是 LOG.md）
3. **push 之后才 ping**，不 ping 未推状态
4. **lock_scope 显式写进 NOW.md**

### 4 个检查点
- **CP-1**：改 `SPEC.md` 前必须先 ack（防单方面改契约）
- **CP-2**：`NOW.md` 显示「等用户验收」时需主动确认
- **CP-3**：冲突 30 分钟无应升级到用户
- **CP-4**：`lock_scope` 同时占用的先到先得规则

### cockpit/ 文件分工

| 文件 | 写什么 | 不要写什么 |
|---|---|---|
| `SPEC.md` | API 契约 / 数据模型 / 长期决策 | 流水 / 当下进度 |
| `NOW.md` | 当前 active_owner / lock_scope / phase（覆盖式） | 历史 / 契约 |
| `LOG.md` | 已发生事实（commit hash / 验证命令）（append-only） | 契约 / 当下任务 |
| `COORDINATION.md` | 协作协议（拷自模板） | 一次写定不再改 |

详细协议见 [SKILL.md](./SKILL.md)。

## 开发记录

这个 skill 经过两轮严格的 TDD 流程做出来：
- 第一轮：用 superpowers:writing-skills 的 RED-GREEN-REFACTOR 写出 baseline
- 第二轮：用 darwin-skill 的三角分离（Claude 定方案 / Codex 改代码 / 独立 Opus 评分员评分）做了 3 轮优化，分数从 86.4 → 89.5

来源：[`leyanai-station`](https://leyanai.com) 项目 2026-04-27 Claude × Codex 实战。

## License

MIT — 随便用，改了也行，做了改进欢迎 PR。
