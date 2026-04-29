---
name: cockpit-spec
status: draft
---

# 项目规格 SPEC

> **稳定契约。架构 / API / 字段 / 长期决策的正源。**
> **不写流水（流水写 LOG.md）。不写当下进度（写 NOW.md）。**
> 改动需要双方同步；改完在 LOG.md 注一笔"已改 SPEC §X.Y"。

## 1. 角色分工

| 角色 | tmux pane | 职责 |
|---|---|---|
| Claude | `<session>:0.0` | 例：方案 + 前端 |
| Codex | `<session>:0.1` | 例：审方案 + 后端 |
| Hermes / 调度员（可选）| `<session>:0.2`（按 `pane_current_path` 定位，编号不固定）| 只读巡检 / 翻译 / 调度 / ping；**默认不改业务代码**，必要时只最小更新 `cockpit/NOW.md` / `LOG.md` |

**按项目调整，但一定在这里写死，避免互抢活。**
**协作正源只有 `cockpit/` 一份**，Hermes 不另开 `hermes/` / `dispatch/` 等平行目录；SPEC.md 不许 Hermes 单方面改（CP-1）。

## 2. 数据模型 / API 契约

> 所有契约写这里。不要把契约写进 LOG.md。

### 2.1 例：`GET /api/weather`

| 字段 | 类型 | 说明 |
|---|---|---|
| `temp_c` | number | 摄氏温度 |
| `desc` | string | 天气描述 |
| `updated_at` | ISO8601 | 数据时间 |
| `cached` | bool | 是否命中缓存 |

错误：HTTP 502 + `{"error": string}`。
缓存策略：TTL 10 分钟。

## 3. 部署与基础设施

> 服务器、域名、systemd unit、nginx 配置位置。

## 4. 验证手段

> 测试命令、E2E 验证、health check。

## 5. 已决议（不再讨论）

- [YYYY-MM-DD] 决策内容（提议人 / 反对意见 / 最终选择）

## 6. Open questions（等拍板）

- Q1: ...

## 7. Changelog

- [YYYY-MM-DD] 初版
