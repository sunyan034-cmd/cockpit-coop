# Cockpit 协作日志

> 双 AI 协作的事实流水。**只 append，不删改历史**。
> **不写契约（契约写 SPEC.md）。不写当下任务（当下写 NOW.md）。**
> 用来追溯，不当当前状态源——当前状态看 NOW.md。

## 格式约定

- 新条目 append 到底部
- 每条头部：`### [YYYY-MM-DD HH:MM] [角色] 标题`
- 角色：`[Claude]` / `[Codex]` / `[大哥]`
- 修改 SPEC.md 后在本日志注一笔（"已改 SPEC §X.Y"）

## 一条标准记录长这样

```markdown
### [2026-04-27 11:00] [Claude] M3 brief 截断按句号

- 改了什么：MAX_BRIEF_CHARS 200→280，截断按句号边界
- 验证：`python3 -m pytest tests/test_brief.py -q` → 24 passed
- 部署：`./deploy.sh` 通过
- commit: 4091640
- 已改 SPEC §3.2 brief 字段说明
```

---

## YYYY-MM-DD

### [HH:MM] [角色] 标题

- ...
