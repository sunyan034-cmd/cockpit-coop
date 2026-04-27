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
| 其他 | `<可空>` | `<可空>` |

## tmux session

- session 名：`<project-session>`
- Claude pane：`<session>:0.0`
- Codex pane：`<session>:0.1`
- 通知对象：`<按项目实际填写>`

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
