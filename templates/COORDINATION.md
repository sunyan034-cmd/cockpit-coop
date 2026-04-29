---
name: cockpit-coordination
status: active
---

# 双 AI 协作项目配置

> 完整协议见 `~/.claude/skills/cockpit-coop/SKILL.md`，本文件只列项目可定制项。
> 规则变更优先改 skill；本文件只记录本项目差异。

## 角色分工

| 角色 | Agent / pane | 默认范围 |
|---|---|---|
| Claude | `<agent/pane>` | `<frontend 或指定目录>` |
| Codex | `<agent/pane>` | `<backend 或指定目录>` |
| Hermes / 调度员（可选）| `<session>:0.2` 或同项目任意第三 pane | 只读巡检 / 翻译 / 调度 / ping；**不改业务代码**，必要时只最小更新 `cockpit/NOW.md` / `LOG.md` 并在 LOG 注 `[Hermes]` |
| 其他 | `<可空>` | `<可空>` |

> **协作正源只有 `cockpit/` 一份**：Hermes 不准建 `hermes/` / `dispatch/` 等平行状态目录；SPEC.md 不许 Hermes 单方面改（沿用 CP-1）。

## tmux session

- session 名：`<project-session>`
- Claude pane：`<session>:0.0`
- Codex pane：`<session>:0.1`
- Hermes pane（可选）：同 session 内 `tmux split-window` 起的第三 pane；**编号不固定**，按 `pane_current_path` + 项目名定位（见下）
- 通知对象：`<按项目实际填写>`

## Hermes 定位规则（模糊项目名也能找到目标 pane）

大哥喊"调度 image" / "翻译 Codex" / "巡检项目"时，Hermes 不依赖固定 pane 编号：

```bash
PROJ_PATH=$(pwd)   # Hermes 启动时所在的项目目录
tmux list-panes -a -F \
  "#{session_name}:#{window_index}.#{pane_index} #{pane_current_command} #{pane_current_path}" \
  | awk -v p="$PROJ_PATH" '$3==p'
```

只要工作目录匹配（或项目名是路径子串），就能锁定 Claude / Codex 所在 pane，再按铁律 1 发 `[Hermes] ...` 短 ping。

## lock_scope 命名规则

- 默认写相对路径或稳定模块名：`web/`、`services/api/`、`docs/api.yaml`
- 需要更细时写到文件或接口：`frontend: weather page`、`api: GET /weather`
- 本项目额外前缀 / 禁用范围：`<可空>`

## 特殊例外条款

- `<例外 1：例如某目录只能由一个 agent 改>`
- `<例外 2：例如某命令只能由 human 执行>`
- 无例外时写：`无`

## 本项目常用入口

- 主要验证命令：`<npm test / pytest / make test>`
- 部署或发布窗口：`<可空>`
- 关键外部系统：`<可空>`

## 维护规则

- 本文件只写项目差异，不复制通用协议。
- 通用协议变化时，改 `~/.claude/skills/cockpit-coop/SKILL.md`。
- 新 agent 首次进入先读本文件，再回到 `SKILL.md` 查完整规则。
