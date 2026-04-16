# 任务索引（TASKS.md）

> **维护者**：CTO（`/CLAUDE.md` 角色）。Sub-agent 不直接修改本文件，仅在交付描述中建议状态变更，由 CTO 实际更新。
>
> **作用**：
> 1. 提供所有任务的**快速索引**（单文件一眼扫）
> 2. 任务**详细简报**不在本文件，而在 `docs/tasks/active/T-XXX.md` 和 `docs/tasks/archive/YYYY-MM/T-XXX.md`
> 3. 避免单文件无限膨胀——活跃任务超过 10 个仍保持可读性

---

## 状态定义

| 标记 | 状态 | 含义 |
|---|---|---|
| 📋 | `TODO` | 已创建未派发 |
| 🚧 | `DOING` | 已派发给 sub-agent，执行中 |
| 🔍 | `REVIEW` | Sub-agent 已交付，待 CTO Review |
| 🔁 | `REWORK` | Review 不通过已打回，执行中（记录轮次） |
| ⏸️ | `BLOCKED` | 阻塞，备注写明原因与解除条件 |
| ✅ | `DONE` | 验收通过（迁移到 archive） |
| 🗑️ | `CANCELLED` | 撤销（迁移到 archive） |

## 任务类型

| 类型 | 承接 sub-agent |
|---|---|
| `DEV_BACKEND` | backend-dev |
| `DEV_FRONTEND` | frontend-dev |
| `OPS` | ops |
| `QA` | qa |
| `SPIKE` | 任一（探索型任务，不进主干） |
| `HUMAN_CHANGE` | — （人类手动提交，CTO 事后登记） |

---

## 文件组织

```
docs/
├── TASKS.md                 ← 本文件：索引与状态一览
└── tasks/
    ├── README.md            ← tasks 目录使用规则
    ├── TEMPLATE.md          ← 任务简报模板
    ├── active/              ← 状态非 DONE / CANCELLED 的任务
    │   ├── T-012.md
    │   └── T-013.md
    └── archive/             ← DONE / CANCELLED 的任务（按月分目录）
        ├── 2026-04/
        │   ├── T-001.md
        │   └── T-002.md
        └── 2026-05/
```

**迁移规则**（由 CTO 维护）：
- 任务创建 → `tasks/active/T-XXX.md`（从 `TEMPLATE.md` 复制）
- 状态变化 → 更新 `active/T-XXX.md` 的状态栏 + 本文件的表格
- 验收 / 撤销 → 移动到 `tasks/archive/YYYY-MM/T-XXX.md` + 本文件表格状态改 DONE/CANCELLED

---

## 活跃任务（状态非 DONE / CANCELLED）

| ID | 任务 | 类型 | 承接 | 关联 | 状态 | 创建 | 更新 | 详情 |
|----|------|------|------|------|------|------|------|------|

*（当前无活跃任务。首次创建任务后，在此表追加一行，并创建 `tasks/active/T-XXX.md`。）*

---

## 最近归档（最多显示最近 10 条，详情在 archive 目录）

| ID | 任务 | 结论 | 归档日期 | 位置 |
|----|------|------|---------|------|

*（暂无归档任务）*

> 完整历史按月在 `tasks/archive/YYYY-MM/` 下。

---

## Changelog（本索引文件的结构变更，不记录每个任务）

| 日期 | 变更 |
|---|---|
| 2026-04-17 | 改造为纯索引文件，任务详情迁至 `docs/tasks/active/` 与 `docs/tasks/archive/` |
| 2026-04-17 | 初始化任务日志结构 |
