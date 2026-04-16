# 功能清单（FEATURES）

> 本文档记录项目中**所有功能点**及其**当前状态**。
> 任何功能变化都必须同步更新此文档（详见项目根 `CLAUDE.md` §13）。
> 废弃条目不删除，标记为 `🗑️ DEPRECATED` 并注明原因与替代方案。

---

## 状态定义

| 标记 | 状态 | 含义 |
|---|---|---|
| 📋 | `PLANNED` | 已规划，尚未开始开发 |
| 🚧 | `IN_PROGRESS` | 正在开发中 |
| ✅ | `DONE` | 已完成且通过验证 |
| ⏸️ | `ON_HOLD` | 暂停，需备注原因 |
| 🗑️ | `DEPRECATED` | 已废弃，需备注原因与替代方案 |
| 🧪 | `EXPERIMENTAL` | 实验性，未定型可能大改 |

---

## 功能总览

### 用户与鉴权（`user` / `auth` 模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-001 | 用户注册 | 🗑️ | - | 业务决策：改为零门槛匿名身份，不做注册。替代方案见 F-002 |
| F-002 | 匿名会话初始化（`POST /session/init`） | 📋 | 2 | 按 deviceId 幂等创建/续签；首次自动分配昵称；返回 token + userId + deviceId + nickname + isNewUser |
| F-003 | 用户登出 | 🗑️ | - | 业务决策：匿名身份无登出概念，关闭标签即离开。仅客户端清 localStorage 即可 |
| F-004 | 管理员封禁（`POST /admin/ban`） | 📋 | 2 | 按 deviceId / IP / 二者组合封禁，含时长；立即下发 `KICK` 并断连 |
| F-005 | 单点登录 | 🗑️ | - | 业务决策：匿名身份无 SSO 概念，不同 deviceId 天然是不同用户 |
| F-006 | JWT 鉴权过滤器 | 📋 | 2 | HTTP + WS 握手共用；除验 token 外还校验 session 与封禁状态 |
| F-007 | 匿名昵称自动生成 | 📋 | 2 | 格式 `游客` + 6 位随机数字；首次 session init 时分配 |
| F-008 | 用户修改昵称（`PATCH /user/nickname`） | 📋 | 2 | 格式校验（2~20 字符，可配）；若在房间内冲突则拒绝（`NICKNAME_IN_USE`） |
| F-009 | IP 限流：匿名身份创建 | 📋 | 2 | 每 IP 每小时 20 次，可配；超限返回 `RATE_LIMITED_IP` |

### 位置（`location` 模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-010 | 房间地理点注册 | 📋 | 4 | Redis GEO `GEOADD` |
| F-011 | 附近房间查询 | 📋 | 4 | `GEOSEARCH` 按半径 |
| F-012 | 房间地理点移除 | 📋 | 4 | 房间关闭时调用 |

### 聊天室（`chatroom` 模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-020 | 按位置加入房间（命中即加入） | 📋 | 4 | 调用 `RoomSelectStrategy`；加入时若昵称与房内成员冲突自动加后缀 `(n)`，存入 `nc:room:{roomId}:display-names` |
| F-021 | 按位置创建房间（半径内无房间） | 📋 | 4 | 中心点固定为创建者位置 |
| F-022 | 选房策略：最近房间 | 📋 | 4 | `NearestRoomStrategy` 默认实现 |
| F-023 | 选房策略抽象（可扩展点） | 📋 | 4 | `RoomSelectStrategy` 接口 |
| F-024 | 用户主动离开房间 | 📋 | 4 | 同时从 members 与 display-names Hash 中移除 |
| F-025 | 用户断连自动离开 | 📋 | 5 | Netty `channelInactive` 触发 |
| F-026 | 空房间自动关闭 | 📋 | 4 | 成员数归零立即关；一并删除 display-names |
| F-027 | 房间成员查询 | 📋 | 4 | 内部 API，不直接暴露给客户端 |
| F-028 | 加入流程并发控制 | 📋 | 4 | Redisson 分布式锁 |
| F-029 | 孤儿房间兜底清理 | 📋 | 7 | 每分钟定时扫描，分布式锁抢占 |

### IM 通用抽象（`im-core` 模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-030 | `Message` 消息原语 | 📋 | 3 | 含 conversationType |
| F-031 | `Conversation` 会话抽象 | 📋 | 3 | PRIVATE / GROUP / CHATROOM |
| F-032 | `MessageSender` 发送接口 | 📋 | 3 | |
| F-033 | `AckHandler` ack 处理接口 | 📋 | 3 | CHATROOM 为单次 ACK（OK/FAILED，语义=已持久化）；PRIVATE/GROUP 预留扩展位 |
| F-034 | `PushChannel` 推送通道抽象 | 📋 | 3 | Gateway 提供实现 |
| F-035 | 消息过滤器接口 `MessageFilter` | 📋 | 3 | 敏感词钩子，默认空实现 |

### 消息（`message` 模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-040 | 消息发送入口（persist + broadcast 两段分离） | 📋 | 6 | `MessageService.persist` 同步 + `MessageService.broadcast` 异步 |
| F-041 | 消息入库（MongoDB 按月分集合） | 📋 | 6 | **同步写**，成功后才 ACK；serverMsgId 由 Gateway 传入 |
| F-042 | 消息广播（Redis Pub/Sub） | 📋 | 6 | `nc:ch:room:{roomId}`；ACK 之后异步触发，失败仅记日志 |
| F-043 | 消息长度校验 | 📋 | 6 | 默认 500，可配 |
| F-044 | 发送频率限流 | 📋 | 6 | 每用户每秒 1 条，Redis INCR + EXPIRE |
| F-045 | 成员资格校验 | 📋 | 6 | 非成员不能发 |
| F-046 | 敏感词过滤（钩子） | 📋 | 6 | 通过 F-035 接口预留，本期不实现具体规则 |
| F-047 | 单次 ACK 协议（OK / FAILED） | 📋 | 6 | 语义="已持久化不丢"；入库成功后才回 ACK；广播失败不影响 ACK |
| F-049 | serverMsgId 生成（Gateway 侧） | 📋 | 5 | Gateway 雪花 ID，传给 Message 模块复用，保证 ACK 与广播 ID 一致 |
| F-048 | 历史消息查询（客户端） | 🗑️ | - | 业务决策：不对客户端提供，仅风控/审查用 |

### Gateway WebSocket（`gateway` 模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-050 | WebSocket 握手鉴权 | 📋 | 5 | Token 走 query `?token=...` |
| F-051 | 连接生命周期管理 | 📋 | 5 | `ChannelGroup` + 本地 Map |
| F-052 | 心跳检测与超时断开 | 📋 | 5 | 默认 90s 无读超时 |
| F-053 | 本地 `userId → Set<Channel>` 映射 | 📋 | 5 | 支持同用户多端在线 |
| F-054 | Redis 频道订阅与推送分发 | 📋 | 5 | 订阅 `nc:ch:room:*` |
| F-055 | 协议编解码（JSON） | 📋 | 5 | 消息类型：`MSG` / `ACK` / `MSG_IN` / `PING` / `PONG` / `KICK` / `ROOM_CLOSED` / `ERROR` |
| F-056 | 监听 `UserKickedEvent` 关闭连接 | 📋 | 5 | 跨节点通过 Redis 频道；关闭前先下发 `KICK` 帧 |
| F-057 | 断连触发 `ChatroomService.leave` | 📋 | 5 | `channelInactive` 回调 |
| F-058 | 消息链路编排（Gateway 侧） | 📋 | 5/6 | 收 `MSG` → 生成 serverMsgId → 同步 `persist` → 回 `ACK` → 异步 `broadcast` |
| F-059 | WebSocket `KICK` / `ROOM_CLOSED` 下发 | 📋 | 5 | 服务端主动通知前端断开原因 |

### 基础设施（`common` / `infrastructure` 模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-060 | 统一返回体 `Result<T>` + 错误码 | 📋 | 1 | |
| F-061 | 业务异常 + 全局异常处理 | 📋 | 1 | |
| F-062 | 分布式锁抽象 `DistributedLock` | 📋 | 1 | Redisson 实现 |
| F-063 | 领域事件发布抽象 `DomainEventPublisher` | 📋 | 1 | Spring Event 实现，预留 Kafka 替换位 |
| F-064 | 统一 ID 生成（雪花） | 📋 | 1 | |
| F-065 | Redis 模板封装（统一前缀 `nc:`） | 📋 | 1 | |

### 运维与质量（跨模块）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-070 | 日志规范（SLF4J + MDC traceId） | 📋 | 7 | HTTP + WS 全链路 traceId |
| F-071 | ArchUnit 架构规则测试 | 📋 | 7 | 模块依赖方向、internal 封闭 |
| F-072 | 健康检查端点 | 📋 | 1 | `/actuator/health` |
| F-073 | 并发加入房间测试 | 📋 | 4 | 验证 F-028 |
| F-074 | 端到端联调测试 | 📋 | 6 | 两客户端同房间互发 |

### 前端（Vue）

| ID | 功能 | 状态 | Phase | 备注 |
|---|---|---|---|---|
| F-080 | 会话初始化（首次/回访） | 📋 | 7 | 启动时读 localStorage 中 deviceId，调 `/session/init` 获取 token；成功后持久化 deviceId + token |
| F-081 | 位置获取与上报 | 📋 | 7 | H5 Geolocation API |
| F-082 | WebSocket 连接与重连 | 📋 | 7 | |
| F-083 | 聊天室消息收发与展示 | 📋 | 7 | 显示名以 `MSG_IN.nickname`（后端下发的房内显示名）为准 |
| F-084 | 心跳保活 | 📋 | 7 | |
| F-085 | 昵称修改界面 | 📋 | 7 | 调 `PATCH /user/nickname`；处理 `NICKNAME_INVALID` / `NICKNAME_IN_USE` 错误码 |
| F-086 | 封禁提示界面 | 📋 | 7 | 收到 `BANNED` / `KICK:BANNED` 时展示不可重试的终止态 |

---

## 待决策 / 待讨论

> 尚未拍板、需要后续确认的点。确认后迁移到正式功能清单。

| 主题 | 描述 | 待确认人 |
|---|---|---|
| 房间创建速率限制 | 是否需要限制单用户短时间内频繁新建房间（防刷）？ | - |
| 多端消息同步 | 同一 userId 多端登录，消息是否每端都推？（目前默认：都推） | - |
| 地理位置准确度 | 前端上报的位置是否需要做抗作弊校验？ | - |
| UI 库选型 | 前端 Element Plus / Naive UI / Vant / 无库，待用户拍板 | - |
| deviceId 生成源 | 前端 UUID 即可，还是引入浏览器指纹（fingerprintjs）增强追踪？前者简单，后者对反小号更有效但有隐私争议 | - |

---

## 变更日志（Changelog）

> 格式：`YYYY-MM-DD | 变更描述 | 影响的 Feature ID`
> 最新的放最上面。

| 日期 | 变更 | Feature ID |
|---|---|---|
| 2026-04-16 | 产品方向转向"零门槛匿名身份"：F-001/003/005 废弃；F-002 改为 `/session/init` 匿名会话；F-004 改为 `/admin/ban` 封禁；F-006 扩充；新增 F-007（昵称生成）/F-008（昵称修改）/F-009（IP 限流）；前端 F-080 从登录页改为会话初始化，新增 F-085（昵称修改界面）/F-086（封禁提示）；F-020/024/026 备注补充房内显示名 Hash 的维护；API_SPEC 升级 v0.2.0 | F-001~F-009、F-020/024/026、F-080~F-086 |
| 2026-04-16 | 待决策项「用户昵称变更同步」已敲定（已发送消息不回改，新消息用新显示名），从待决策表移除；新增「deviceId 生成源」 | - |
| 2026-04-16 | ACK 语义调整为"单次 ACK + 已持久化不丢"；`MessageService` 拆分为 `persist`（同步）+ `broadcast`（异步）；serverMsgId 改由 Gateway 生成 | F-033, F-040, F-041, F-042, F-047 |
| 2026-04-16 | 新增 F-049（Gateway 侧 serverMsgId 生成）、F-058（Gateway 消息链路编排）、F-059（`KICK` / `ROOM_CLOSED` 下发） | F-049, F-058, F-059 |
| 2026-04-16 | WS 协议消息类型定稿（`MSG` / `ACK` / `MSG_IN` / `PING` / `PONG` / `KICK` / `ROOM_CLOSED` / `ERROR`）并同步至 `API_SPEC.md` | F-055 |
| 2026-04-16 | 待决策项「消息 ID 生成位置」已敲定（Gateway 生成），从待决策表移除；新增「UI 库选型」 | - |
| 2026-04-16 | 初始创建功能清单，录入 Phase 1~7 全部计划功能 | F-001 ~ F-084 |
| 2026-04-16 | F-048（历史消息查询）按业务决策标记为 DEPRECATED | F-048 |

---

## Feature ID 分配约定

- `F-0XX`：用户/鉴权（001-009）
- `F-01X`：位置（010-019）
- `F-02X`：聊天室（020-029）
- `F-03X`：IM Core（030-039）
- `F-04X`：消息（040-049）
- `F-05X`：Gateway（050-059）
- `F-06X`：基础设施（060-069）
- `F-07X`：运维与质量（070-079）
- `F-08X`：前端（080-089）
- `F-09X`：预留未来（090+）

新增功能在对应段内顺序分配下一个可用编号，**不复用废弃 ID**。
