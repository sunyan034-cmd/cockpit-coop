---
name: cockpit-now
date: YYYY-MM-DD
status: 一句话当前状态
---

# Cockpit · Now

> **当前接力状态。**每次 owner 切换或阶段变化时**覆盖式更新**（不是 append）。
> **不写流水（流水写 LOG.md）。不写契约（契约写 SPEC.md）。**

## 👀 大哥看这里（人话进度）

> 一句话：今天 [谁] 干完了 [什么]，下一步 [谁] 接 [什么]，等大哥拍 [什么]（如有）。
>
> 例：今天 Claude 切完前端天气源、Codex 接后端缓存审核中。明天大哥看下 leyanai.com 首页天气有没有显示。

## Current

- latest_commit: `<hash>`（以 `git log -1` 为准）
- active_owner: `Claude` | `Codex` | `none`
- phase: 当前一句话状态

## Lock scope（互不重叠才能并行）

| Owner | Scope | 状态 |
|---|---|---|
| Claude | `web/index.html` 前端骨架 | done @ <hash> / in_progress |
| Codex | `services/api/` 后端 | next（接 Claude draft，stash@{0}）|

## 接续点（下次进来从这开始）

1. 第一步：...
2. 第二步：...
3. 等大哥拍：...

## Open questions（需要对方回答）

- Q: ...

## Ping rule

tmux 短 ping，<80 字符，必带文档指针：

```
[Codex] done <hash>; next=Claude; read cockpit/NOW.md
```

push 后才 ping。长说明全部写本文件或 LOG.md。
