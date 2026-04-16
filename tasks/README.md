# 任务目录（Tasks）

> **维护者**：CTO（`/CLAUDE.md` 角色）。Sub-agent 不直接修改本目录下的文件，仅在交付描述中建议状态变更，由 CTO 实际更新。
>
> **作用**：
> 1. 追踪所有派发给 sub-agent 的任务状态
> 2. `active/*.md` 是 CTO 跨 session 的"工作记忆外挂"——会话切换后，CTO 通过读取这些文件无缝续跑
> 3. 任务与 `FEATURES.md` 的 Feature ID、`API_SPEC.md` 的章节关联

---

## 目录结构

```
docs/tasks/
├── README.md            ← 本文件：索引、统计、状态定义
├── TEMPLATE.md          ← 任务简报模板（新建任务时复制）
├── active/              ← 所有未完成任务（TODO / DOING / REVIEW / REWORK / BLOCKED）
│   ├── T-012.md
│   ├── T-013.md
│   └── ...
└── archive/             ← 已完成/已撤销任务，按月归档
    ├── 2026-04/
    │   ├── T-001.md
    │   └── T-002.md
    └── 2026-05/
```

**迁移规则**：
- 任务创建 → 放 `active/`
- 任务 DONE 或 CANCELLED → 月末批量移到 `archive/YYYY-MM/`（或立即移动，视任务量）
- 永不删除任务文件；归档是冻结而非遗忘

---

## 状态定义

| 标记 | 状态 | 含义 |
|---|---|---|
| 📋 | `TODO` | 已创建未派发 |
| 🚧 | `DOING` | 已派发给 sub-agent，执行中 |
| 🔍 | `REVIEW` | Sub-agent 已交付，待 CTO Review（含 code-reviewer 前置） |
| 🔁 | `REWORK` | Review 不通过已打回，执行中（记录轮次） |
| ⏸️ | `BLOCKED` | 阻塞，备注写明原因与解除条件 |
| ✅ | `DONE` | 验收通过 |
| 🗑️ | `CANCELLED` | 撤销，备注原因 |

## 任务类型

| 类型 | 承接 sub-agent | 说明 |
|---|---|---|
| `DEV_BACKEND` | backend-dev | 后端代码实现、单测、修 bug |
| `DEV_FRONTEND` | frontend-dev | 前端代码实现、单测 |
| `OPS` | ops | 容器、CI/CD、部署、迁移 |
| `QA` | qa | 集成测试、回归测试 |
| `SPIKE` | 任一 dev | 调研、原型、可行性验证。产出**不进主干**，只作信息 |
| `HUMAN_CHANGE` | — | 人类变更记录（不派发，仅归档说明） |

---

## 活跃任务索引

> 更新规则：任务状态变化时必须同步此表。

| ID | 标题 | 类型 | 承接 | 状态 | 关联 Feature | 最后更新 | 简报 |
|----|------|------|------|------|-----------|---------|------|
| *（当前无活跃任务）* |

---

## 本月归档

| ID | 标题 | 类型 | 结论 | 归档 |
|----|------|------|------|------|
| *（当前月份无归档任务）* |

---

## 统计（CTO 启动时更新）

- 📋 TODO: 0
- 🚧 DOING: 0
- 🔍 REVIEW: 0
- 🔁 REWORK: 0
- ⏸️ BLOCKED: 0
- ✅ DONE（本月）: 0
- 🗑️ CANCELLED（本月）: 0

**健康信号**：
- 返工 (REWORK) 比例 > 30% → 任务简报质量不足或 sub-agent 规范需强化
- BLOCKED 任务停留 > 3 天 → 必须主动提醒用户决策
- 活跃任务 > 10 → 考虑是否 scope 过大需要拆分

---

## Changelog

| 日期 | 变更 |
|---|---|
| 2026-04-17 | 从单文件 `TASKS.md` 迁移到目录结构；建立 active / archive 分区 |
