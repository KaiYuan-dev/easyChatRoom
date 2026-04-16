# ADR-XXXX: <简短标题>

- **状态**: Proposed | Accepted | Deprecated | Superseded by ADR-YYYY | Rejected
- **日期**: YYYY-MM-DD
- **决策者**: CTO（代表全队）
- **相关**:
  - Supersedes: ADR-XXXX（可选，若替代了旧决策）
  - Superseded by: ADR-YYYY（可选，若被新决策替代——此字段在后续更新时才填）
  - Relates to: ADR-XXXX / FEATURES F-XXX / API_SPEC §X.Y

---

## 背景（Context）

<描述当时的处境与触发本次决策的问题。
 读者是一个 6 个月后的新人——不要依赖"大家都懂"的隐含信息。
 包含：
 - 业务背景（这个决策服务于什么需求）
 - 约束（已有的技术栈、人力、时间）
 - 痛点（不决策的话会怎样）>

---

## 考虑过的方案（Considered Options）

### 方案 A：<名称>

<简短描述，占 1~3 行>

**优点：**
- ...

**缺点：**
- ...

### 方案 B：<名称>

<同上>

### 方案 C：<若有>

<同上>

---

## 决策（Decision）

**选择方案 X**。

<用 1~3 句话清晰阐述决策内容。具体到"做什么"，不要只写"我们决定改进性能"这种模糊表述。>

---

## 理由（Rationale）

<为什么选这个方案而不是其他？关键权衡是什么？
 至少回答：
 - 为什么选了 A 而否决 B？
 - B 的缺点对我们不可接受在哪？
 - A 的缺点我们如何缓解？>

---

## 后果（Consequences）

<这是最重要的部分。坦率列出本决策带来的一切影响，好的坏的都写。>

### 积极后果
- ...

### 消极后果 / 代价
- ...

### 未来需要关注的风险
- ...

---

## 实施要点（可选）

<若决策需要特定的实施步骤或约束，在此列出。例如"XX 中间件必须配置为集群模式"之类。>

---

## 历史（可选）

<如果本 ADR 在后续被修订过状态（例如从 Proposed 到 Accepted，再到 Superseded），在这里按时间顺序记录。正文不改，只在此追加。>

- YYYY-MM-DD: 状态从 Proposed 改为 Accepted（<简短原因>）
- YYYY-MM-DD: 状态改为 Superseded by ADR-YYYY
