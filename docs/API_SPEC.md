# API 契约文档（API_SPEC）

> **维护方**：仅后端维护。前端只读。
> **作用**：前后端接口契约唯一真相源。前端代码必须严格对齐本文档，不得自行假设或发明接口。
> 本文档的任何变更均可能影响前端代码，变更后须在对话中提示："前端请同步 `API_SPEC.md` §X"。

---

## 版本信息

- **Last Updated**: 2026-04-16
- **当前版本**: v0.2.0（未实现）
- **最近变更摘要**: 去除注册/登录，改为零门槛匿名身份（deviceId 持久化）；新增 `/session/init`、`/user/nickname`、`/admin/ban`；调整错误码与 KICK reason

---

## 1. 通用约定

### 1.1 基础 URL

| 类型 | URL | 说明 |
|---|---|---|
| HTTP Base | `http://<host>:8080/api` | REST 接口 |
| WebSocket | `ws://<host>:8081/ws?token=<JWT>` | 长连接，token 走 query |

### 1.2 HTTP 请求规范

- Content-Type：`application/json; charset=utf-8`
- 鉴权 Header：`Authorization: Bearer <JWT>`
- 时间戳字段统一用 Unix 毫秒（`long`）

### 1.3 统一响应结构

所有 HTTP 接口返回统一结构：

```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... }
}
```

- `code = 0` 表示成功；非 0 表示失败，具体见 §5 错误码表
- `data` 失败时可能为 `null`
- HTTP 状态码：200（业务正常或业务异常）、401（未鉴权 / Token 无效）、403（封禁）、429（IP 限流）、500（系统异常）

### 1.4 鉴权流程

1. `POST /session/init` 首次访问创建匿名身份并获取 JWT；回访携带 `deviceId` 获得稳定身份
2. HTTP 请求：Header 带 `Authorization: Bearer <JWT>`
3. WebSocket：连接时 URL 附带 `?token=<JWT>`
4. 若 JWT 过期或被封禁 → HTTP 返回 401/403，WebSocket 握手被拒或主动断开

---

## 2. HTTP 接口

### 2.1 `POST /session/init` — 初始化匿名会话

**无需鉴权**。首次访问与回访皆调用此接口。

**请求体**：
```json
{
  "deviceId": "d_xxx_or_null"
}
```

- `deviceId`：前端 localStorage 中持久化的值；首次访问传 `null` 或省略
- 客户端 IP 由服务端从请求头获取（用于限流与封禁检查）

**成功响应**：
```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "token": "eyJhbGci...",
    "userId": 123,
    "deviceId": "d_snow_xxx",
    "nickname": "游客384721",
    "isNewUser": true,
    "expiresAt": 1745000000000
  }
}
```

**字段说明**：
- `deviceId`：若请求未带，服务端生成并返回；前端必须持久化到 localStorage
- `isNewUser`：true 表示此 deviceId 首次访问（新建了 user 记录）
- `nickname`：若 `isNewUser=true` 为自动分配；回访返回当前已存昵称

**可能错误码**：
- `RATE_LIMITED_IP`：同 IP 创建匿名身份超限（默认 20 次/小时）
- `BANNED`：deviceId 或 IP 在封禁名单
- `SYSTEM_ERROR`

**备注**：此接口**幂等**——同一个 deviceId 多次调用总是返回同一 userId（昵称可能已被用户修改）。

---

### 2.2 `PATCH /user/nickname` — 修改昵称

**鉴权**：需要

**请求体**：
```json
{ "nickname": "新昵称" }
```

**成功响应**：
```json
{
  "code": 0,
  "msg": "ok",
  "data": { "nickname": "新昵称" }
}
```

**可能错误码**：
- `NICKNAME_INVALID`：格式不合法（长度、非法字符）
- `NICKNAME_IN_USE`：用户当前在某房间中，且新昵称与房内其他成员冲突
- `SYSTEM_ERROR`

**备注**：昵称长度默认 2~20（可配 `auth.nickname.min-length` / `max-length`）。不在任何房间时修改永远成功；在房间时会做冲突检查。

---

### 2.3 `POST /room/join` — 按位置加入/创建房间

**鉴权**：需要

**请求体**：
```json
{
  "lng": 116.404,
  "lat": 39.915
}
```

**成功响应**：
```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "roomId": "r_abc123",
    "centerLng": 116.404,
    "centerLat": 39.915,
    "memberCount": 5,
    "createdAt": 1745000000000,
    "isNewlyCreated": false
  }
}
```

**字段说明**：
- `centerLng` / `centerLat`：房间中心点（固定，不会因加入新成员而改变）
- `isNewlyCreated`：true 表示本次调用创建了新房间，false 表示加入现有房间

**可能错误码**：
- `LOCATION_INVALID`：经纬度非法
- `SYSTEM_ERROR`：后端系统异常

**备注**：获取房间信息后，前端需建立 WebSocket 连接（若尚未建立），并开始监听该 `roomId` 的消息。服务端会根据 JWT 识别用户所在房间，前端无需显式 `subscribe`。

---

### 2.4 `POST /admin/ban` — 封禁（管理接口，前端暂不对接）

> 仅运营 / 管理后台使用。

**鉴权**：需要（管理员权限机制本期未实现，接口先留）

**请求体**：
```json
{
  "deviceId": "d_xxx",
  "ip": "1.2.3.4",
  "durationSeconds": 3600,
  "reason": "ABUSE"
}
```

**字段说明**：
- `deviceId` / `ip`：至少填一个；可同时填（二维度独立，任一命中即封禁）
- `durationSeconds`：-1 表示永久
- `reason` 枚举：`ABUSE` / `SPAM` / `POLICY_VIOLATION` / `OTHER`

**成功响应**：
```json
{ "code": 0, "msg": "ok", "data": null }
```

**行为**：封禁生效后，若目标 deviceId 当前在线，服务端立即下发 `{type:"KICK", reason:"BANNED"}` 并关闭连接。IP 维度封禁依赖后续握手检查生效。

---

## 3. WebSocket 协议

### 3.1 握手

- URL：`ws://<host>:8081/ws?token=<JWT>`
- 握手失败：服务端立即关闭连接，close code 使用标准 `4401`（鉴权失败）、`4403`（无权限）

### 3.2 消息封装

所有 WebSocket 消息为 JSON 文本帧，必含字段：

```json
{ "type": "<消息类型>", ...其他字段 }
```

`type` 枚举见下表。

### 3.3 消息类型总表

| type | 方向 | 说明 |
|---|---|---|
| `PING` | C → S | 心跳请求，无 payload |
| `PONG` | S → C | 心跳响应 |
| `MSG` | C → S | 客户端发送消息 |
| `ACK` | S → C | 消息发送结果（单次 ACK） |
| `MSG_IN` | S → C | 广播给房间成员的新消息 |
| `KICK` | S → C | 服务端主动踢出（登录被顶号、被管理员踢等）|
| `ROOM_CLOSED` | S → C | 当前房间已关闭 |
| `ERROR` | S → C | 协议级错误（非业务错误）|

### 3.4 客户端 → 服务端

#### 3.4.1 `MSG` — 发送消息

```json
{
  "type": "MSG",
  "clientMsgId": "c_uuid_xxx",
  "roomId": "r_abc123",
  "content": "hello world"
}
```

字段说明：
- `clientMsgId`：前端生成的 UUID，用于关联 ACK
- `content`：消息内容，最长 **500** 字符（UTF-8 字符数）

#### 3.4.2 `PING` — 心跳

```json
{ "type": "PING" }
```

前端每 30 秒发送一次。90 秒无读超时服务端会主动断开。

---

### 3.5 服务端 → 客户端

#### 3.5.1 `ACK` — 消息发送结果（核心）

**语义约定**：ACK 表示"**消息已持久化，服务端承诺不丢**"，而非"网络已收到"。每条 `MSG` 有且仅有一次 `ACK`。

**成功**：
```json
{
  "type": "ACK",
  "clientMsgId": "c_uuid_xxx",
  "serverMsgId": "m_snow_yyy",
  "ts": 1745000000000,
  "status": "OK"
}
```

**失败**：
```json
{
  "type": "ACK",
  "clientMsgId": "c_uuid_xxx",
  "status": "FAILED",
  "code": "RATE_LIMITED",
  "msg": "sending too fast"
}
```

**常见失败 code**：`CONTENT_TOO_LONG` / `RATE_LIMITED` / `NOT_ROOM_MEMBER` / `ROOM_NOT_FOUND` / `SYSTEM_ERROR`

#### 3.5.2 `MSG_IN` — 房间内广播消息

```json
{
  "type": "MSG_IN",
  "serverMsgId": "m_snow_yyy",
  "roomId": "r_abc123",
  "userId": 456,
  "nickname": "Bob",
  "content": "hi",
  "ts": 1745000000000
}
```

**注意**：
- 发送者自己也会收到自己消息的 `MSG_IN`。前端可用 `serverMsgId` 去重（ACK 时已拿到自己消息的 `serverMsgId`）
- `nickname` 为**房内显示名**（可能与用户原昵称不同——加入时若冲突会被自动加后缀 `(2)` 等）。前端直接使用此字段渲染，不需要做昵称映射
- 若用户在房间中修改昵称，已发送消息的 `nickname` 不回改；之后的新消息会使用新的显示名

#### 3.5.3 `KICK` — 被踢下线

```json
{
  "type": "KICK",
  "reason": "LOGIN_FROM_OTHER_DEVICE"
}
```

**reason 枚举**：
- `BANNED`：deviceId 或 IP 被封禁
- `KICKED_BY_ADMIN`：被管理员主动断开（非封禁）
- `TOKEN_EXPIRED`：token 过期
- `SESSION_INVALID`：session 失效（Redis 中不存在）

收到后服务端会立即关闭连接。前端根据 reason 做不同提示。

#### 3.5.4 `ROOM_CLOSED` — 房间已关闭

```json
{
  "type": "ROOM_CLOSED",
  "roomId": "r_abc123",
  "reason": "EMPTY"
}
```

**备注**：在当前实现中，空房间关闭时正在房间中的用户应该已经全部离开，此消息主要用于兜底场景。

#### 3.5.5 `PONG` — 心跳响应

```json
{ "type": "PONG" }
```

#### 3.5.6 `ERROR` — 协议级错误

```json
{
  "type": "ERROR",
  "code": "PROTOCOL_INVALID",
  "msg": "unknown message type"
}
```

仅在协议层错误时使用，业务错误走 `ACK`。

---

## 4. 限流与约束

| 项 | 当前值 | 说明 |
|---|---|---|
| 单条消息最大字符数 | 500 | UTF-8 字符数 |
| 单用户发送频率 | 每秒 1 条 | 超限返回 `ACK.code=RATE_LIMITED` |
| 昵称长度 | 2~20 字符 | 可配 `auth.nickname.min-length` / `max-length` |
| 同 IP 创建匿名身份 | 每小时 20 次 | 可配 `auth.session-init.ip-limit-per-hour` |
| WebSocket 心跳间隔 | 客户端 30s | |
| WebSocket 读超时 | 服务端 90s | 超时断连 |
| JWT 有效期 | 86400 秒（24 小时） | 可配 |
| 加入房间的搜索半径 | 默认 1000 米 | 可配 `chatroom.join-radius-meters` |

---

## 5. 错误码表

### 5.1 HTTP / 业务错误码

| code | 含义 | 建议前端行为 |
|---|---|---|
| `0` | 成功 | - |
| `AUTH_TOKEN_INVALID` | Token 无效或过期 | 清理本地 token，重走 `/session/init` |
| `BANNED` | deviceId 或 IP 被封禁 | 提示"账号或设备被限制"，不要自动重试 |
| `RATE_LIMITED_IP` | 同 IP 创建匿名身份超限 | 提示稍后再试，不要轮询 |
| `NICKNAME_INVALID` | 昵称格式不合法 | 按提示规则重新输入 |
| `NICKNAME_IN_USE` | 当前在某房间内，新昵称与房内成员冲突 | 提示用户换一个 |
| `LOCATION_INVALID` | 经纬度非法 | 检查定位权限与上报值 |
| `ROOM_NOT_FOUND` | 房间不存在 | 离开房间界面 |
| `CONTENT_TOO_LONG` | 消息超长 | 前端应做前置校验 |
| `RATE_LIMITED` | 发消息限流 | 提示"发送太快" |
| `NOT_ROOM_MEMBER` | 不是房间成员 | 重新加入房间 |
| `SYSTEM_ERROR` | 系统异常 | 通用失败提示 |

### 5.2 WebSocket close code

| code | 含义 |
|---|---|
| `1000` | 正常关闭 |
| `4401` | 握手鉴权失败（JWT 无效） |
| `4403` | 无权限 |
| `4409` | deviceId 冲突或被封禁（服务端主动关闭，先发 `KICK` 再关） |

---

## 6. 变更日志（Changelog）

> 格式：`YYYY-MM-DD | 变更摘要`
> 最新的放最上面。

| 日期 | 变更 |
|---|---|
| 2026-04-16 | v0.2.0：去除注册/登录；新增 `/session/init`（匿名身份 + deviceId 持久化）、`/user/nickname`、`/admin/ban`；调整 KICK reason 与错误码（删除 `AUTH_INVALID_CREDENTIAL` / `AUTH_ACCOUNT_LOCKED`，新增 `BANNED` / `RATE_LIMITED_IP` / `NICKNAME_INVALID` / `NICKNAME_IN_USE`）；限流表补充昵称长度与 IP 创建限额 |
| 2026-04-16 | 初版：鉴权、房间加入、WebSocket 基础协议、错误码表 |

---

## 7. 待定 / 后续版本

- 消息撤回：当前版本不支持
- 图片 / 文件消息：当前版本不支持（仅文本）
- 房间列表查询：用户不可手选房间，接口不提供
- 历史消息查询：业务决策不对客户端提供

如需以上能力，先在 `FEATURES.md` 创建 PLANNED 条目讨论。
