# Task T-XXX: <简短标题>

**状态**: 📋 TODO | 🚧 DOING | 🔍 REVIEW | 🔁 REWORK | ⏸️ BLOCKED | ✅ DONE | 🗑️ CANCELLED
**类型**: DEV_BACKEND | DEV_FRONTEND | OPS | QA | SPIKE | HUMAN_CHANGE
**承接**: backend-dev | frontend-dev | ops | qa | code-reviewer | — (HUMAN_CHANGE)
**关联 Feature**: F-XXX
**创建**: YYYY-MM-DD
**最后更新**: YYYY-MM-DD
**依赖任务**: T-YYY（若有）

---

## Context
<1~3 句话说明背景与必要上下文；引用相关文档章节>

## 目标
<明确说明要产出什么；对应 FEATURES.md 的 Feature ID>

## 范围
- **在范围内**：
  - <具体列出>
- **不在范围内**：
  - <明确排除，防止 scope creep>

## 参考文档
- `<对应 CLAUDE.md>` §X.Y
- `docs/API_SPEC.md` §X.Y（若涉及接口）
- `docs/FEATURES.md` F-XXX
- `docs/ADR/ADR-XXXX-*.md`（若相关）

## 约束
<非功能性要求：性能、并发、兼容性等>
<引用架构铁律的相关条款>

## 相关经验（从 LESSONS.md 引用，无则写"无"）
- `LESSONS.md` §YYYY-MM-DD-id: <一句话摘要>

## 完成定义（DoD）
- [ ] 代码实现符合任务范围
- [ ] 单元测试覆盖（由对应 dev sub-agent 负责）
- [ ] 通过 code-reviewer 自动化检查
- [ ] 文档同步更新（FEATURES.md / 对应 CLAUDE.md，若需要）
- [ ] 提交描述清晰

## 交付格式（sub-agent 返回时）
请在返回时包含：
1. 修改/新建的文件清单
2. 核心设计思路（3~5 行）
3. 自检清单（是否符合约束）
4. 任何假设或未覆盖的点

---

## Spike 专属字段（仅当 Task Type = SPIKE，否则删除本节）
- **调研问题**：<具体问题>
- **时间盒**：<不得超过>
- **可接受的结论**：<含"不可行"作为合法结论>
- **输出形式**：调研报告 / 原型代码（**不进主干**）
- **非预期扩展**：发现其他问题记录但不处理

---

## Review 记录

### 第 1 轮
- **派发时间**: YYYY-MM-DD HH:MM
- **交付时间**: YYYY-MM-DD HH:MM
- **code-reviewer**: PASS / FAIL（详见报告）
- **CTO 业务 Review**: PASS / REWORK / BLOCK
- **备注**: <关键反馈>

### 第 2 轮（若有返工）
...

---

## 最终结论（完成后填）
- **状态**: ✅ DONE / 🗑️ CANCELLED
- **总轮次**: N 轮
- **关键学到**: <如需记入 LESSONS.md，在此提示>
- **归档位置**: `archive/YYYY-MM/T-XXX.md`
