# 附近聊天室 · 后端 CLAUDE.md

**读者**：本仓库的 Claude Code。
**作用**：本文档是后端开发的**强制规范**。产出必须符合本文档；冲突时以本文档为准。
**协同文档**：
- `../docs/FEATURES.md`——功能清单，前后端共享，你维护后端条目
- `../docs/API_SPEC.md`——接口契约，**由你维护**，前端只读


---

## 0. 会话启动必执行

每次新会话开启时，按顺序执行：

1. 读取本文档
2. 读取 `../docs/FEATURES.md`
3. 读取 `../docs/API_SPEC.md`

完成前不响应用户任务。任一文档缺失，告知用户并停止。

**每次产出代码前，你必须确认：**

- 产出符合 §3（架构铁律）和 §7（开发规范）
- 对外接口有变更 → 同步更新 `API_SPEC.md`，并在回复中提示"前端请同步 `API_SPEC.md` §X"
- 功能状态有变更 → 同步更新 `FEATURES.md` 条目 + Changelog
- 新架构决策或用户给出新规范 → 同步更新本文档对应章节
- 文档更新与代码变更同提交，或作为紧邻的前/后置 commit

详细维护规则见 §13。

**硬性禁止项：**

- 禁止杜撰用户需求。需求模糊时询问用户，不得自行发挥
- 禁止跳过文档更新直接写代码
- 禁止使用 `synchronized` / `ReentrantLock` 保护共享状态
- 禁止跨模块 `@Transactional`
- 禁止从一个模块 import 另一个模块的 `internal` 包
- 禁止在代码里留下 TODO 而不开 FEATURES 条目追踪

---

## 1. 项目定位

**业务**：基于地理位置的临时公共聊天室。**零门槛，无需注册/登录**，用户打开网页即可使用。按位置决定加入附近已存在的聊天室或创建新聊天室；空房间立即关闭。

**核心约束**：
- **匿名身份**：首次访问签发匿名 JWT，基于持久化 `deviceId`（前端存 localStorage）识别回访用户
- **自动昵称**：后端首次签发时自动分配昵称（如 `游客384721`），用户可修改
- **同房间昵称不重名**：加入房间时若昵称冲突则自动加后缀；用户主动改名在房内冲突时拒绝
- 位置驱动，用户不可主动选房
- 同一地理区域同一时刻只存在一个房间
- 房间临时存在，无人即关
- 消息持久化用于风控审查，**不对客户端提供历史**
- **反滥用**：按 `deviceId` + `ip` 封禁；同一 IP 创建匿名身份有速率限额

**平台范围**：本期仅支持 Web 浏览器端。移动 App 暂不考虑，但 `deviceId` 抽象必须允许未来接入设备级标识。

**部署形态**：当前单体，代码必须按分布式多实例前提编写，未来可平滑拆分为微服务。

---

## 2. 技术栈

| 层次 | 选型 | 使用约束 |
|---|---|---|
| 语言 / 框架 | Java 17 + Spring Boot 3.x | |
| 长连接 | Netty 4.x (WebSocket) | 独立端口，共用 Spring 容器 |
| HTTP | Spring MVC | REST：登录、加入房间 |
| 缓存 / 地理 / 广播 | Redis（GEO + Pub/Sub + Session） | Spring Data Redis + Redisson |
| 分布式锁 | Redisson | **所有**跨请求共享资源锁都走这里 |
| 消息持久化 | MongoDB | 按月分集合 |
| 业务数据库 | MySQL 8.x | 用户等结构化数据 |
| ORM | MyBatis-Plus | MySQL 访问 |
| 构建 | Maven | 多模块 |
| 鉴权 | JWT + Redis Session | JWT 无状态 + Session 支持踢人 |

若用户要求新增或替换依赖，先更新本章，再改 `pom.xml`。

---

## 3. 架构铁律

**以下规则不可违反。发现违反立即修正，不征求确认。**

### 3.1 分布式就绪

即使当前单体部署，所有代码按多实例前提编写。

**禁止：**

1. **禁止用本地同步锁保护跨请求共享状态**：不得使用 `synchronized`、`ReentrantLock`、`AtomicXxx` 保护"多节点需一致"的状态。并发控制一律走 Redisson 分布式锁，或 Redis 原子操作（INCR / SETNX / Lua）。
   - 例外：纯进程内、无跨实例一致性需求的场景（如一个工具类内部 ConcurrentHashMap 做本地只读缓存），允许使用 JDK 并发工具。判定标准："这个状态是否需要跨实例一致？"

2. **禁止跨模块共享本地事务**：`@Transactional` 仅限保护**单个模块内部**的数据库操作。跨模块一致性走领域事件 + 最终一致性。
   - 反例：`user` 模块 Service 事务中调 `message` 模块落库 —— 禁止。
   - 正确：user 事务提交后发 `UserRegisteredEvent`，message 模块异步订阅处理。

3. **禁止跨模块直接引用另一模块的 Service 实现类 / Repository / Domain**：跨模块只能 import 目标模块的 `api` 包。

4. **禁止在内存中持有权威状态**：房间成员、在线用户、房间元信息等"多人可见 + 多节点需同步"的状态必须放 Redis。进程内仅允许持有**连接级本地资源**（如 Netty Channel 与 userId 的映射）。

5. **禁止 `@Scheduled` 在所有节点重复执行**：定时任务要么通过 Redisson 分布式锁抢占只一个节点执行，要么设计成幂等可并行。

### 3.2 模块化

1. **跨模块通信只走 `api` 包**：每个业务模块对外暴露 `api` 子包，含 Service 接口、DTO、事件定义。其他模块只依赖 `api`，不依赖 `internal`。

2. **跨模块异步通信优先用领域事件**：当前实现用 Spring `ApplicationEventPublisher`，必须封装在 `DomainEventPublisher` 抽象后，未来可替换为 Kafka / RocketMQ。

3. **循环依赖不可出现**：Maven 构建期即拦截。若 A、B 相互需要，处理方式：
   - 引入底层模块 C，A、B 都依赖 C
   - 用事件反转依赖
   - 共享契约提至 `common` 或独立 `contract` 模块

4. **一个模块一个职责**：描述模块职责时若需用"和"、"并且"连接多个职责，必须拆分。

### 3.3 抽象度判定

- **接口抽象触发条件**：能具体列出 2 个以上真实变化方向
- **仅一个变化方向**：不抽象，直接实现
- **不会变化**：用最易读的直写方式
- **设计模式使用前提**：能具体回答"不用会有什么问题"，否则不用

**禁止过度设计**。不允许为"未来可能"引入工厂、策略、模板方法。

---

## 4. 模块划分

### 4.1 模块清单

```
nearby-chat/                          (父 pom)
├── nearby-chat-common/               通用工具（无业务、无框架依赖）
├── nearby-chat-infrastructure/       基础设施抽象（Redis / 分布式锁 / 事件）
│
├── nearby-chat-user/                 用户域
├── nearby-chat-auth/                 认证域（JWT + Session + 踢人）
├── nearby-chat-location/             位置域（Redis GEO 封装）
├── nearby-chat-chatroom/             聊天室域
├── nearby-chat-im-core/              IM 通用抽象（Message / Sender / Ack）
├── nearby-chat-message/              消息持久化与广播
├── nearby-chat-gateway/              Netty WebSocket 入口
│
└── nearby-chat-app/                  启动模块（main + 装配）
```

### 4.2 依赖方向

**只允许向下依赖：**

```
                         app
                          │
        ┌──────┬──────┬───┴───┬──────┬─────────┐
     gateway message chatroom auth  user    (其他)
        │      │      │       │      │
        └──────┴──┬───┴───────┴──────┘
                  │
              im-core
                  │
            ┌─────┴─────┐
            │           │
         location    (其他)
                │
         infrastructure
                │
             common
```

**强制规则：**

- `common` 不依赖任何业务模块
- `infrastructure` 只依赖 `common`
- `im-core` 不依赖任何业务模块（通用 IM 能力，不感知"附近聊天室"业务）
- 业务模块间**不互相依赖**；需要对方能力走事件或底层共享
- `app` 是唯一能看到所有模块的位置，负责装配

### 4.3 模块内部包结构（所有业务模块统一）

示例（chatroom）：

```
com.xxx.nearbychat.chatroom
├── api/                          对外暴露，其他模块只能 import 这里
│   ├── ChatroomService.java      接口
│   ├── dto/
│   │   ├── JoinRoomCommand.java
│   │   └── RoomInfo.java
│   └── event/
│       └── RoomCreatedEvent.java
│
├── internal/                     内部实现，禁止被外部 import
│   ├── domain/
│   ├── repository/
│   ├── service/
│   └── strategy/
│
├── web/                          HTTP Controller
│   └── ChatroomController.java
│
└── config/
    └── ChatroomProperties.java
```

**强制规则：**

- 其他模块 import 本模块时，**只允许 `api.*`**
- `internal` 下的类尽量 `package-private`，让编译器挡
- 使用 ArchUnit 测试验证（Phase 7 必须实现）

---

## 5. 各模块对外契约

### 5.1 `common`

产出：`Result<T>` 返回体、错误码枚举、`BusinessException` / `SystemException`、ID 生成工具、时间工具。
约束：无框架或中间件依赖。

### 5.2 `infrastructure`

产出：
- `RedisTemplate` 封装（统一 key 前缀、序列化）
- `DistributedLock` 抽象 + Redisson 实现
- `DomainEventPublisher` 抽象 + Spring Event 实现

### 5.3 `user`

职责：匿名用户档案（`userId` + `deviceId` + 昵称）。**本项目无注册/登录账号体系**。

对外：
- `UserService.createAnonymous(CreateAnonymousCommand) -> UserInfo`：按 deviceId 创建或返回已有档案
- `UserService.getById(userId) -> UserInfo`
- `UserService.getByDeviceId(deviceId) -> Optional<UserInfo>`
- `UserService.updateNickname(userId, newNickname)`：仅校验格式；业务层冲突检查由调用方负责（见 §6.5）

事件：`UserCreatedEvent`、`NicknameChangedEvent`

内部：首次创建时调用 `NicknameGenerator` 自动分配（格式 `游客` + 6 位随机数字）。

### 5.4 `auth`

职责：匿名会话签发（JWT）、封禁管理、鉴权校验。

对外：
- `AuthService.initSession(InitSessionCommand) -> SessionResult`：按 deviceId 初始化会话（新建或续签）；返回 `{token, userId, deviceId, nickname, isNewUser}`
- `AuthService.validate(token) -> AuthContext`：解析 JWT，校验 session 存活 + 未被封禁
- `AuthService.ban(BanCommand)`：封禁 deviceId / IP / 二者组合，指定时长；发 `UserBannedEvent`
- `AuthService.isBanned(deviceId, ip) -> boolean`

事件：`UserSessionInitEvent`、`UserBannedEvent`

JWT payload：`{userId, deviceId, sessionId, iat, exp}`

**封禁语义**（详见 §6.4）：
- 维度：deviceId / ip / 二者（任一命中即拒绝）
- 时长：秒级，支持永久（约定值或无 TTL）
- 存储：Redis
- 握手时检查；已连接用户触发事件后断连

### 5.5 `location`

对外：
- `LocationService.addRoomPoint(roomId, lng, lat)`
- `LocationService.removeRoomPoint(roomId)`
- `LocationService.searchNearbyRooms(lng, lat, radiusMeters, limit) -> List<NearbyRoom>`

无业务语义，只提供地理能力。

### 5.6 `chatroom`

对外：
- `ChatroomService.joinByLocation(userId, lng, lat) -> RoomInfo`
- `ChatroomService.leave(userId, roomId)`
- `ChatroomService.getRoomInfo(roomId) -> RoomInfo`
- `ChatroomService.listMembers(roomId) -> Set<Long>`

事件：`RoomCreatedEvent`、`RoomClosedEvent`、`UserJoinedRoomEvent`、`UserLeftRoomEvent`

选房策略通过 `RoomSelectStrategy` 接口扩展。

### 5.7 `im-core`

通用 IM 抽象，不感知业务。

核心类型：`Conversation`（PRIVATE / GROUP / CHATROOM）、`Message`、`MessageSender`、`AckHandler`、`PushChannel`、`MessageFilter`

语义约定：
- `CHATROOM`：单次 ACK（OK / FAILED），语义为"已持久化不丢"
- `PRIVATE` / `GROUP`：保留 per-receiver `DELIVERED` 扩展位（本期不实现）

### 5.8 `message`

对外（两段分离，配合 Gateway 单次 ACK）：
- `MessageService.persist(PersistMessageCommand)`：**同步**持久化，失败抛业务异常；Gateway 在 ACK 前必须完成
- `MessageService.broadcast(BroadcastCommand)`：**异步**广播到 Redis 频道；Gateway 在 ACK 后触发，失败仅记日志

约束：
- `serverMsgId` 由 Gateway 传入，本模块不生成
- 持久化到 MongoDB，按月分集合
- Redis Pub/Sub 广播到 `nc:ch:room:{roomId}`

### 5.9 `gateway`

Netty WebSocket 入口，职责：
- 握手鉴权（JWT）
- 连接生命周期管理（`ChannelGroup` + `userId → Set<Channel>` 本地映射）
- 协议编解码（JSON，详见 `API_SPEC.md`）
- `PushChannel` 实现（本节点推送）
- 订阅 `nc:ch:room:*` 频道，推送给本节点的在线成员
- 订阅 `UserKickedEvent`，下发 `KICK` 帧并关闭该用户所有 Channel
- 消息链路编排（见 §6.5）

### 5.10 `app`

职责：`main`、`application.yml`、装配所有模块。
约束：不含业务逻辑。

---

## 6. 关键技术设计

### 6.1 选房策略

```java
public interface RoomSelectStrategy {
    /**
     * 候选房间已按距离升序、且都在允许半径内。
     * 返回 empty 表示新建房间。
     */
    Optional<String> select(double lng, double lat, List<NearbyRoom> candidates);
}
```

默认实现 `NearestRoomStrategy`：候选非空取第一个，否则 empty。

扩展策略（例如 `PopularityStrategy`、`TopicMatchStrategy`）通过 Spring `@ConditionalOnProperty` 切换或配置文件指定 bean 名称。

### 6.2 房间归属（中心点固定）

- 新建时以创建者位置为中心，永久固定
- 加入流程：
  1. 用户上报 `(lng, lat)`
  2. `LocationService.searchNearbyRooms(lng, lat, n)` 返回半径内候选
  3. `RoomSelectStrategy.select(...)` 决定加入哪个或新建
  4. 新建则 `addRoomPoint`；加入则更新成员集
- **并发控制**：加入 / 创建流程必须在 Redisson 分布式锁下执行，lock key 用 geohash 前缀粗分桶防止同位置并发创建两个房间

### 6.3 房间生命周期

- 加入：`SADD nc:room:{roomId}:members {userId}`
- 离开（主动 / 断连）：
  1. `SREM nc:room:{roomId}:members {userId}`
  2. 检查 `SCARD nc:room:{roomId}:members`
  3. 若为 0：删元信息 + 删 GEO 点 + 发 `RoomClosedEvent`
- 兜底定时任务（Phase 7）：每分钟扫描，清理长时间无活动的孤儿房间；分布式锁保证单节点执行

### 6.4 匿名会话 + 封禁

**会话初始化（首次 / 回访）：**

1. 前端调 `POST /session/init`，body 含 `deviceId`（可能为 null 或陌生值）
2. IP 限流检查：`nc:ratelimit:session:ip:{ip}`，`INCR + EXPIRE`，超限返回 `RATE_LIMITED_IP`
3. 若请求未带 deviceId → 后端生成 UUID 作为 deviceId
4. 封禁检查：`AuthService.isBanned(deviceId, ip)` → 命中返回 `BANNED`
5. 查 user：`UserService.getByDeviceId(deviceId)`
   - 命中 → 复用已有 userId 与昵称
   - 未命中 → `createAnonymous(deviceId)` 生成 userId + 自动昵称
6. 生成 `sessionId = UUID`，`SET nc:session:{sessionId} userId EX <jwtExpire>`
7. `SADD nc:user-sessions:{userId} {sessionId}`
8. 签发 JWT（payload：`{userId, deviceId, sessionId, iat, exp}`）
9. 返回 `{token, userId, deviceId, nickname, isNewUser}`

**前端负责**：将 `deviceId` 与 `token` 持久化到 `localStorage`。下次访问带上 `deviceId` 再调 `/session/init`。

**鉴权（HTTP + WS 握手）：**
1. 解 JWT 拿 `sessionId` + `deviceId`
2. `GET nc:session:{sessionId}`，不存在或 userId 不一致 → 401 / close(4401)
3. `isBanned(deviceId, ip)` → 命中 → 403 / close(4403)

**封禁（管理接口）：**

`POST /admin/ban`，body 字段：
- `deviceId`（可选）
- `ip`（可选）
- `durationSeconds`（-1 表示永久）
- `reason`（枚举）

至少填一个维度。处理流程：
1. 写入 Redis：`nc:ban:device:{deviceId}` 和 / 或 `nc:ban:ip:{ip}`，含 TTL
2. 若有 deviceId：查找对应 userId，删除其所有 session，发 `UserBannedEvent{userId, reason}`
3. 所有 gateway 节点订阅事件，先下发 `{type:"KICK", reason:"BANNED"}` 再关闭本节点该 userId 的 Channel

**IP 维度封禁**（`deviceId` 为空，仅 IP 封禁）：
- 无法精确找到在线用户，依赖握手检查生效
- 已连接的用户在下次重连或心跳超时后被拒

**IP 创建身份限流**（反刷号）：
- `nc:ratelimit:session:ip:{ip}` INCR，TTL 1 小时
- 超过阈值（默认 20 次/小时，配 `auth.session-init.ip-limit-per-hour`）返回 `RATE_LIMITED_IP`

### 6.5 消息链路（核心）

**ACK 语义**：`ACK.status=OK` 表示"**消息已持久化，服务端承诺不丢**"，不是"网络已收到"。ACK 必须在入库成功之后发出。整个链路**仅一次 ACK**。

```
Client
  │ 1. {type:"MSG", clientMsgId, roomId, content}
  ▼
Gateway
  │ 2. 生成 serverMsgId（雪花）+ ts
  │ 3. 同步调 MessageService.persist(...)
  ▼
Message
  │ 4. 校验（长度 / 限流 / 成员 / 敏感词钩子）
  │ 5. 同步写 MongoDB
  │ 6. 成功返回，或抛业务异常
  ▼
Gateway
  │ 7. 回 {type:"ACK", clientMsgId, serverMsgId, ts, status:"OK"}
  │    ← 此时消息已持久化
  │ 8. 异步调 MessageService.broadcast(...)
  ▼
Message
  │ 9. Redis PUBLISH nc:ch:room:{roomId}
  │
  │ 所有 Gateway（含本节点）订阅该频道
  │ 10. 推给本节点该房间在线成员
  ▼
Other Clients {type:"MSG_IN", serverMsgId, ...}
```

**关键约束：**

- `serverMsgId` 由 Gateway 生成，不由 Message 模块生成
- 步骤 3~6 同步，MongoDB 写入延迟同机房 < 10ms
- 步骤 8~10 异步，广播失败只记 error 日志，不回流给客户端；消息仍在 DB 中，接受实时到达损失（业务不提供历史）
- 校验失败：步骤 5 前拒绝，回 `{type:"ACK", clientMsgId, status:"FAILED", code, msg}`
- 系统异常：回 FAILED + `code=SYSTEM_ERROR`，不暴露细节

**协议字段以 `API_SPEC.md` 为准**。本节示例仅示意。

### 6.6 消息约束

- **具体数值权威位置**：`docs/API_SPEC.md` §4（限流与约束表）
- **默认值的配置位置**：§8 `application.yml`（`chatroom.message.max-length` / `chatroom.message.rate-limit-per-sec`）
- 单条文本限流：按 `API_SPEC.md §4` 定义的字符数上限；Redis `INCR` + `EXPIRE 1` 实现发送频率限流
- 成员资格：非房间成员不能发
- 敏感词：通过 `MessageFilter` 接口预留空实现

### 6.7 连接与会话管理

- Netty `ChannelGroup` 持所有活跃连接
- 本地 `ConcurrentHashMap<Long, Set<Channel>>` 维护 userId → channels（多端在线）
- Channel attribute：`userId`、`sessionId`、`roomId`
- `channelInactive` 回调：
  1. 调 `ChatroomService.leave(userId, roomId)`
  2. 清本地映射

### 6.8 昵称冲突处理

用户层有稳定的 `nickname`（存在 `user` 表），但 Redis 中的**房间成员显示名**可能与之不同以保证同房间不重名。

**数据结构：**

`nc:room:{roomId}:display-names` Hash，`userId -> roomDisplayName`

**加入房间时**（自动加后缀）：
1. 读取用户 `nickname`
2. 扫描 `room:{roomId}:display-names` 的 value 集合
3. 若冲突：生成 `"{nickname}({n})"`，n 从 2 开始自增直到不冲突
4. `HSET room:{roomId}:display-names {userId} {roomDisplayName}`
5. 广播消息时用 `roomDisplayName`

**用户改名时**（在房内冲突则拒绝）：
1. `UserService.updateNickname(userId, newNickname)` 仅校验格式与长度
2. `ChatroomService` 订阅 `NicknameChangedEvent`：
   - 若该 userId 在房间中，检查 `room:{roomId}:display-names` 是否冲突
   - 冲突 → 抛业务异常回滚改名；**此处需在事务边界内同步完成**，见下文
   - 无冲突 → 更新 `room:{roomId}:display-names` 中该 userId 的 value
3. 不在任何房间 → 仅更新 user 表

**关于"改名同步冲突校验"的实现约束：**
- 跨模块一致性不能用事务（§3.1）
- 实现方式：`updateNickname` API 入口**同步**调用 `ChatroomService.checkNicknameAvailable(userId, newNickname)` 作为前置校验，通过后再改 user 表
- 这是同步查询，不是跨模块事务；不违反 §3.1
- 若校验通过但改表后房间状态变化（例如改名瞬间有同名用户加入），以"改名成功"为准，下一个加入者会自动加后缀

**离开房间时**：`HDEL room:{roomId}:display-names {userId}`；房间关闭时整个 Hash 随 meta 一起删除。

---

## 7. 开发规范

### 7.1 命名

| 类型 | 规则 | 示例 |
|---|---|---|
| 包名 | 全小写，单数 | `chatroom`、`message` |
| 类名 | 大驼峰 | `ChatroomService` |
| 接口 | 不加 `I` 前缀 | `MessageSender` |
| 实现类 | 单实现加 `Impl` 后缀；多实现按语义命名 | `ChatroomServiceImpl`；`NearestRoomStrategy` |
| DTO | 加用途后缀 | `JoinRoomCommand`、`RoomInfo`、`MessageAck` |
| 事件 | 过去分词 + `Event` | `RoomCreatedEvent` |
| 常量 | 全大写下划线 | `MAX_MESSAGE_LENGTH` |
| 方法 | 小驼峰，动词开头 | `joinByLocation` |
| Redis Key | 冒号分层，前缀 `nc:` | `nc:room:{roomId}:members` |
| 配置前缀 | 模块名开头 | `chatroom.join-radius-meters` |

### 7.2 DTO 约定

- `Command`：入参，描述动作（`JoinRoomCommand`）
- `Query`：查询入参（`NearbyRoomQuery`）
- `Info` / `View` / `DTO`：出参
- `Event`：领域事件
- **禁止用 domain 对象跨模块传递**；api 层只暴露 DTO，domain 留在 internal

### 7.3 异常

- 业务异常抛 `BusinessException(ErrorCode, msg)`，全局 handler 转 HTTP 响应
- 系统异常抛 `SystemException`，记 error 日志
- 禁止 catch `Exception` 吞异常
- WebSocket 错误通过 `ACK.status=FAILED` 返回，不断连（鉴权 / 协议级错误除外）

### 7.4 日志

- 使用 SLF4J
- 关键业务入口打 info，参数脱敏
- 异常打 error + 堆栈
- 禁止 `e.printStackTrace()`
- 所有请求带 `traceId`（MDC），WebSocket 会话生成 sessionId 全程带

### 7.5 事务

- `@Transactional` 仅限单模块内 DB 操作
- 事务中禁止 RPC、消息发送、调远程
- 跨模块一致性走事件 + 最终一致性

### 7.6 测试

- 每个 Service 核心方法有单元测试
- 每个 `api` 接口有集成测试
- 关键并发场景必须有并发测试（见 F-073）
- ArchUnit 测试覆盖（Phase 7）：
  - 模块依赖方向
  - `internal` 包禁止被外部引用
  - 禁止使用 `synchronized` 保护共享状态

---

## 8. 配置项

`application.yml` 关键项：

```yaml
server:
  port: 8080

netty:
  ws-port: 8081
  boss-threads: 1
  worker-threads: 0              # 0 = CPU * 2
  idle-read-seconds: 90

jwt:
  secret: ${JWT_SECRET}
  expire-seconds: 86400

auth:
  session-init:
    ip-limit-per-hour: 20       # 同 IP 每小时最多创建匿名身份数
  nickname:
    min-length: 2
    max-length: 20

chatroom:
  join-radius-meters: 1000       # n 米
  idle-timeout-seconds: 300      # 兜底扫描
  message:
    max-length: 500
    rate-limit-per-sec: 1

spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
    mongodb:
      uri: ${MONGO_URI}
  datasource:
    url: ${MYSQL_URL}
    username: ${MYSQL_USER}
    password: ${MYSQL_PWD}
```

新增配置项时更新本章。

---

## 9. 数据模型

### 9.1 MySQL

**user**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 雪花 ID（= userId）|
| device_id | VARCHAR(64) UK | 前端 localStorage 持久化的设备标识 |
| nickname | VARCHAR(64) | 昵称，可修改 |
| created_at | DATETIME | |
| updated_at | DATETIME | |

索引：`UNIQUE(device_id)`（支撑回访查找）

**无注册账号表。无 username / password_hash 字段。**

### 9.2 MongoDB

**message**（按月分集合 `message_YYYYMM`）
```
{
  _id: ObjectId,
  serverMsgId: "m_xxx",
  roomId: "r_xxx",
  userId: 123,
  content: "...",
  geoHash: "9qqj...",
  ip: "1.2.3.4",
  createdAt: ISODate
}
```

索引：`(roomId, createdAt)`、`(userId, createdAt)`、`createdAt`

### 9.3 Redis Key

| Key | 类型 | 说明 | TTL |
|---|---|---|---|
| `nc:session:{sessionId}` | String | userId | JWT 有效期 |
| `nc:user-sessions:{userId}` | Set | sessionId 集合 | 无 |
| `nc:device:{deviceId}` | String | userId（反查用） | 无 |
| `nc:ban:device:{deviceId}` | String | 封禁原因 | 按封禁时长 |
| `nc:ban:ip:{ip}` | String | 封禁原因 | 按封禁时长 |
| `nc:ratelimit:session:ip:{ip}` | String INCR | 匿名身份创建限流 | 3600s |
| `nc:geo:rooms` | GEO | 所有房间点 | 无（关闭时清） |
| `nc:room:{roomId}:meta` | Hash | 房间元数据 | 无 |
| `nc:room:{roomId}:members` | Set | 成员 userId | 无 |
| `nc:room:{roomId}:display-names` | Hash | userId → 房内显示名（去重） | 无 |
| `nc:ratelimit:msg:{userId}` | String INCR | 发消息限流 | 1s |
| `nc:lock:room-join:{geoBucket}` | Redisson Lock | 加入 / 创建锁 | 自动 |

### 9.4 Redis Pub/Sub 频道

| Channel | 说明 |
|---|---|
| `nc:ch:room:{roomId}` | 房间内消息广播 |
| `nc:ch:user-kicked` | 用户被踢事件 |
| `nc:ch:room-closed` | 房间关闭事件 |

新增 key / channel 时更新本章。

---

## 10. 分阶段开发计划

**每阶段完成后必须：通过测试、通过 ArchUnit 检查（从 Phase 1 开始）、提交、等待用户 review，再进下一阶段。禁止跨阶段批量产出。**

### Phase 1：骨架 + 架构防线
- 父 pom、所有空模块、依赖关系
- `common`：`Result`、`ErrorCode`、`BusinessException`
- `infrastructure`：Redis 配置、`DistributedLock` 抽象 + Redisson 实现、`DomainEventPublisher` 抽象
- `app`：可启动，`/actuator/health` 返回 200
- **ArchUnit 基础测试（必须，本期落地）**：
  - 模块依赖方向正确（对照 §4.2）
  - `internal` 包禁止被其他模块 import
  - 禁止 `synchronized` / `ReentrantLock` 保护字段（静态规则扫描）
  - CI 中 ArchUnit 失败即构建失败
- **为什么 Phase 1 就上**：§3 铁律若无 CI 执行，实际等于无。每引入一条新铁律（后续阶段）就追加一条 ArchUnit 测试

### Phase 2：匿名身份与鉴权
- `user`：`UserService.createAnonymous` / `getByDeviceId` / `updateNickname`；`NicknameGenerator`
- `auth`：匿名会话初始化（deviceId 持久化识别）、JWT、Session、封禁（deviceId / IP）、IP 限流
- HTTP：`POST /session/init`、`PATCH /user/nickname`、`POST /admin/ban`
- **不实现**：注册、密码登录、登出接口

### Phase 3：IM Core 抽象
- `im-core`：接口定义，仅契约，不实现

### Phase 4：位置与聊天室
- `location`：Redis GEO 封装
- `chatroom`：加入、离开、成员维护、`RoomSelectStrategy` + `NearestRoomStrategy`
- HTTP：`POST /room/join`
- 并发测试验证同位置并发加入
- **追加 ArchUnit 规则**：禁止跨模块 `@Transactional`（若尚未加）

### Phase 5：Gateway
- 握手鉴权、连接管理、心跳
- 协议编解码
- `PushChannel` 实现
- 断连自动 leave
- 订阅 `user-kicked` 频道、下发 `KICK` 帧

### Phase 6：消息
- `message`：`persist` + `broadcast` 两段接口
- 限流、长度、成员、敏感词钩子
- Gateway 订阅 `room:{roomId}` 并推送
- 端到端：两客户端同房间互发

### Phase 7：收尾
- **补全 ArchUnit** 覆盖所有 §3 铁律（Phase 1 已建起骨架；此处确保规则完整）
- 兜底孤儿房间清理定时任务
- 日志、traceId、监控点完整

---

## 11. 架构决策记录（ADR）

| # | 决策 | 原因 |
|---|---|---|
| 1 | Modular Monolith | 起步简单，未来可拆 |
| 2 | 房间中心点固定 | 避免漂移，语义清晰 |
| 3 | Redis GEO 存房间位置 | 性能好，临时语义契合 |
| 4 | 消息存 MongoDB | schema 灵活、水平扩展、时序友好 |
| 5 | 不对客户端提供历史 | 业务决策 |
| 6 | 房间空即关，不用 TTL | 前期量小，TTL 通知性能坑多 |
| 7 | ACK 语义 = 已持久化（单次 ACK） | 先持久化再 ACK，广播异步；聊天室不做 per-receiver delivered |
| 8 | JWT + Redis Session | JWT 规模化 + Session 解决踢人 |
| 9 | 消息长度 500 | 参考同类产品，平衡表达与防刷屏 |
| 10 | IM 抽象为 `im-core` 模块 | 当前不独立，保留未来拆出可能 |
| 11 | serverMsgId 由 Gateway 生成 | Gateway 是客户端唯一入口，ACK 与广播 ID 一致免跨模块往返 |
| 12 | 零门槛匿名身份（无注册/登录） | 降低使用门槛；持久化 deviceId 保留回访体验 |
| 13 | 封禁按 deviceId + IP 维度 | deviceId 精确、IP 兜底反逃逸；二维度独立或组合使用 |
| 14 | 同房间昵称不重名（加入自动加后缀 / 改名冲突拒绝） | 避免房内混淆；不强制全局唯一以减轻改名摩擦 |

新增决策时追加一行。

---

## 12. 文档分工与协作

### 12.1 文档清单

| 文档 | 位置 | 维护方 | 读者 |
|---|---|---|---|
| `CLAUDE.md`（本文档） | `backend/` | 你 | 你 |
| `CLAUDE.md`（前端） | `frontend/` | 前端 Claude Code | 前端 Claude Code |
| `FEATURES.md` | `docs/` | 前后端共同维护 | 前后端共享 |
| `API_SPEC.md` | `docs/` | **你** | 前端只读 |

### 12.2 前后端协作规则

- 前端**不修改** `API_SPEC.md`。新接口或字段需求由你更新 SPEC，前端基于 SPEC 实现
- 你修改对外接口时**必须**同步 `API_SPEC.md`，并在回复中明确提示"前端请同步 `API_SPEC.md` §X"
- `FEATURES.md` 中后端条目（F-001~F-074 等）由你维护；前端条目（F-08X）你不动

### 12.3 交付自检清单

每次交付代码前逐项检查：

1. 是否违反模块依赖方向
2. 是否跨模块 import `internal`
3. 是否用 `synchronized` / `ReentrantLock` 保护共享状态
4. 是否有跨模块 `@Transactional`
5. 命名是否符合 §7.1
6. 对外接口变更是否已同步 `API_SPEC.md`
7. 功能状态变化是否已同步 `FEATURES.md`
8. 相关测试是否补齐

任意一条不满足，立即修正，不交付半成品。

---

## 13. 文档维护协议

### 13.1 必须更新本文档的场景

以下任一触发时立即更新，不等用户提醒：

- 新增模块 / 对外接口 / 领域事件
- 修改现有模块职责或对外接口
- 新增 / 修改技术选型
- 新增 / 修改开发规范
- 用户在对话中提出新的约束、原则或规范
- 新增 ADR
- 修改 Redis Key、配置项、数据模型

### 13.2 不需要更新本文档的场景

- 修 bug、重构内部实现、补单元测试、改注释
- 微调日志、调整依赖版本号
- `internal` 包内部方法重命名（未暴露到 `api`）
- 纯格式化

### 13.3 必须更新 `FEATURES.md` 的场景

任何功能状态变化：

- 用户或讨论中提出新功能 → 新增条目，状态 `📋 PLANNED`
- 某 Phase 开始开发 → 改 `🚧 IN_PROGRESS`
- 通过验证 → 改 `✅ DONE`
- 暂停 → 改 `⏸️ ON_HOLD` + 原因
- 废弃 → 改 `🗑️ DEPRECATED` + 原因 + 替代方案；**不删条目**
- 需求变化 → 更新描述，必要时新增条目

### 13.4 必须更新 `API_SPEC.md` 的场景

- 新增 / 修改 HTTP 接口（路径、方法、入参、响应、错误码）
- 新增 / 修改 WebSocket 消息类型或字段
- 修改鉴权方式
- 修改限流 / 配额等影响前端行为的规则
- 任何会让前端需要改代码的变更

更新后在回复中明确提示："前端请同步 `API_SPEC.md` §X"。

### 13.5 更新执行规则

1. **同提交**：文档变更和代码变更在同一 commit，或作为紧邻的前置 / 后置 commit。禁止累积多轮后批量更新。

2. **更新后在回复中告知用户**：
   - "本次变更已同步 `CLAUDE.md` §5.6"
   - "`FEATURES.md` 新增 F-047，状态 IN_PROGRESS"
   - "`API_SPEC.md` §3.2 新增 WS 消息类型 `KICK`，前端请同步"

3. **模糊时先问**：不确定是否该写入文档 / 写到哪一节，向用户确认，不擅自决定。

4. **用户说"记下这条规范"时**：立即追加到相应文档并 commit，不能只口头应承。

5. **废弃不删除**：保留原条目标记 DEPRECATED + 原因 + 替代方案。

6. **`FEATURES.md` Changelog 必写**：每次更新追加一行：日期 + 变更摘要 + Feature ID。

7. **`API_SPEC.md` 版本头必更新**：修改后更新文档顶部 `Last Updated` 与变更摘要。

8. **章节编号稳定**：尽量只追加，不变动编号。必须调整时在 Changelog 说明迁移路径。

### 13.6 提交前自检

- [ ] 后端外部接口变更 → `CLAUDE.md` §4 / §5 是否同步
- [ ] Redis key / 数据模型变更 → `CLAUDE.md` §9 是否同步
- [ ] HTTP / WS 协议变更 → `API_SPEC.md` 是否同步 + 通知前端
- [ ] 新增功能点 → `FEATURES.md` 是否新增条目
- [ ] 功能状态变化 → `FEATURES.md` 状态 + Changelog 是否同步
- [ ] 用户给出新规范 → 相应 SPEC 章节是否收录

---

## 14. 响应用户任务时的工作流

收到用户任务时按以下顺序执行：

1. **解析任务**：明确任务属于哪个 Phase、涉及哪些模块、是否改动对外接口
2. **检索文档**：本文档相关章节 + `FEATURES.md` 相关条目 + `API_SPEC.md`（若涉及接口）
3. **识别冲突**：任务与现有规范是否冲突
   - 无冲突：执行
   - 有冲突：停下向用户确认，不擅自偏离
4. **产出代码**：符合 §3、§7
5. **更新文档**：按 §13 同步所有相关文档
6. **回复用户**：
   - 说明做了什么
   - 列出文档更新清单（哪些文档的哪些章节）
   - 若涉及前端，明确提示"前端请同步"
   - 指出潜在风险或需要用户决策的遗留问题

**禁止行为：**

- 仅凭记忆响应任务，不读文档
- 产出代码时偏离 §3 铁律
- 完成代码却不更新文档
- 用"以后再补"敷衍文档更新
