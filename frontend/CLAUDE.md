# 附近聊天室项目 · 前端设计规范（FRONTEND CLAUDE.md）

> 本文档是**前端项目**的设计真相源，同时作为 `frontend/CLAUDE.md`。
> 后端有独立的 `CLAUDE.md`（见 `backend/CLAUDE.md`），前后端共享 `FEATURES.md` 与 `API_SPEC.md`。

---

## 0. 启动指令（Claude Code 每次会话必读）

**新会话进入项目时，Claude Code 必须按以下顺序读取文档，无需用户提醒：**

1. 本文档（`CLAUDE.md`）——前端架构与规范
2. `API_SPEC.md`（仓库根目录或 `docs/`）——**前后端接口契约，这是前端唯一的接口信息来源**
3. `FEATURES.md`（仓库根目录或 `docs/`）——当前所有功能点及状态

读完后再回应任务。若任何一份文档缺失，立即告知用户。

**不可逾越的红线：**

- **`API_SPEC.md` 是前端唯一的接口信息来源**。Claude Code 不得自行假设、猜测、发明接口。任何"我记得后端是这样的"都是幻觉。
- **前端不修改 `API_SPEC.md`**。如果需要新接口或改字段，必须停下来，让用户把需求同步给后端，后端更新 `API_SPEC.md` 后再继续。
- **前端只修改 `FEATURES.md` 中前端职责的条目**（F-08X 段为主），不动后端条目的状态。

---

## 1. 项目概述

Vue 3 单页应用，为"附近聊天室"项目提供用户界面。主要能力：

- 账号登录 / 登出
- 获取定位并加入附近聊天室
- 聊天室内消息收发与展示
- WebSocket 长连接与断线重连

详细功能清单见 `FEATURES.md` 的 `F-08X` 段。

---

## 2. 技术栈

| 层次 | 选型 | 说明 |
|---|---|---|
| 框架 | Vue 3 + `<script setup>` | Composition API 优先 |
| 语言 | TypeScript（strict） | 禁用 `any`，必须显式类型 |
| 构建 | Vite | |
| 路由 | Vue Router 4 | |
| 状态管理 | Pinia | 按业务域拆 store |
| HTTP | Axios | 统一封装在 `api/` 层，业务代码不直接调用 axios |
| WebSocket | 原生 `WebSocket` | 封装为可注入的 client，业务代码不直接用原生对象 |
| UI | Element Plus（或其他，待选） | 组件库只在视图层使用 |
| 样式 | SCSS + CSS Module 或 UnoCSS（待选） | |
| 测试 | Vitest + Vue Test Utils | 至少覆盖核心组合函数与状态管理 |
| 代码规范 | ESLint + Prettier + TypeScript strict | pre-commit 必过 |

最终选型由用户拍板；若用户选择与此处不同，更新本章节。

---

## 3. 目录结构

```
frontend/
├── public/
├── src/
│   ├── api/                    # 接口层（封装 HTTP / WS，严格对齐 API_SPEC.md）
│   │   ├── http.ts             # axios 实例 + 拦截器
│   │   ├── ws.ts               # WebSocket 客户端
│   │   ├── session.ts          # /session/init 匿名会话初始化
│   │   ├── user.ts             # /user/nickname 昵称修改
│   │   ├── room.ts             # 房间相关
│   │   └── types.ts            # 由 API_SPEC.md 推导的 DTO 类型
│   │
│   ├── stores/                 # Pinia stores（按业务域）
│   │   ├── session.ts          # deviceId / token / userId / nickname / isBanned
│   │   ├── room.ts             # 当前房间信息
│   │   └── chat.ts             # 消息列表、发送态
│   │
│   ├── composables/            # 可复用的组合式函数
│   │   ├── useWebSocket.ts     # WS 连接管理（封装重连、心跳）
│   │   ├── useGeolocation.ts   # 定位获取
│   │   ├── useSessionInit.ts   # 启动时读 localStorage + 调 /session/init
│   │   └── useMessageSender.ts # 消息发送 + ACK 跟踪
│   │
│   ├── views/                  # 路由级页面
│   │   ├── EntryView.vue       # 入口页：自动 session init + 获取定位 + 跳转
│   │   ├── ChatRoomView.vue
│   │   ├── NicknameView.vue    # 昵称修改页
│   │   └── BannedView.vue      # 被封禁的终止页
│   │
│   ├── components/             # 可复用组件
│   │   ├── MessageList.vue
│   │   ├── MessageInput.vue
│   │   └── MessageBubble.vue
│   │
│   ├── router/
│   ├── utils/                  # 纯工具函数（无副作用）
│   ├── constants/              # 常量、错误码映射
│   ├── styles/
│   ├── App.vue
│   └── main.ts
│
├── tests/
├── CLAUDE.md                   ← 本文档
├── package.json
├── tsconfig.json
├── vite.config.ts
└── .eslintrc.cjs
```

---

## 4. 架构分层原则

自底向上：

```
utils / constants       纯函数、常量
      ↓
api                     接口层，唯一与后端对话的地方
      ↓
composables             业务可复用逻辑
      ↓
stores                  全局状态
      ↓
components              UI 组件
      ↓
views                   页面
```

**硬性约束：**

1. **`views` / `components` 不直接调用 `api`**：必须经过 `composables` 或 `stores` 中转。UI 层保持"哑"，只关心展示与事件。
2. **`api` 层不感知 Pinia / Vue 上下文**：纯函数签名，便于测试与替换。
3. **`composables` 不做 UI 渲染**：只管业务逻辑，返回响应式状态和方法。
4. **`utils` 不依赖 Vue / Pinia / axios**：可以被任何层引用，不引用任何上层。
5. **禁止循环依赖**：store 之间如需互通，通过事件或抽上去的 composable，不互相 import。

---

## 5. 接口层约定（关键）

### 5.1 HTTP 客户端

- 所有请求走 `src/api/http.ts` 的 axios 实例
- 请求拦截器：自动带上 `Authorization: Bearer <token>`（若已初始化会话）
- 响应拦截器：
  - 统一解包 `Result<T>` 结构（字段见 `API_SPEC.md` §1.3）
  - `401` / `AUTH_TOKEN_INVALID` → 清理 localStorage 中的 token（**保留 deviceId**）→ 重新走 `/session/init` → 失败则跳 `EntryView`
  - `403` / `BANNED` → 跳 `BannedView`，**不自动重试**
  - `429` / `RATE_LIMITED_IP` → 展示"稍后再试"，**不自动重试，不轮询**
  - 其他业务错误 → 抛 `BusinessError`，业务层按需捕获
- **接口函数严格对齐 `API_SPEC.md`**。以下是当前版本（v0.2.0）清单，`API_SPEC.md` 每次变更后同步更新：
  - `POST /session/init` — 匿名会话初始化
  - `PATCH /user/nickname` — 修改昵称
  - `POST /room/join` — 按位置加入/创建房间
  - `POST /admin/ban` — 管理接口，前端当前不对接

### 5.2 WebSocket 客户端

- `src/api/ws.ts` 封装单例 `WsClient`，业务代码只面对它
- 能力：
  - 连接、断线自动重连（指数退避）
  - 心跳保活
  - 按 `type` 字段分发消息到订阅者
  - 发送 API + 等待 `ACK`（按 `clientMsgId` 匹配）
- **WebSocket 协议严格以 `API_SPEC.md` 为准**：消息类型、字段，不得自行扩展。

### 5.3 类型定义

`src/api/types.ts` 的类型从 `API_SPEC.md` 推导而来。当 `API_SPEC.md` 更新时：

1. 先更新 `types.ts`
2. 让 TS 编译错误驱动业务代码的修改
3. 同步更新 `FEATURES.md` 若有前端功能受影响

---

## 6. 状态管理

- 按业务域拆 Pinia store，**不做大一统 store**
- store 内部状态用 `ref` / `reactive`，暴露 getter 与 action
- **禁止在 store 里直接调 `api`**：通过 composable 组合。`api` 层是"能做什么"，composable 是"怎么做"，store 是"当前状态"

### 6.1 持久化规则

仅以下字段持久化到 `localStorage`：

| localStorage key | 内容 | 生命周期 |
|---|---|---|
| `nc:deviceId` | 首次 `/session/init` 后由服务端返回的 deviceId | **永久保留**，用户主动清除除外 |
| `nc:token` | 当前有效的 JWT | JWT 过期或失效时清除；`deviceId` 不一起清 |

其他状态（房间信息、消息列表、当前昵称等）**不持久化**，页面刷新即丢失并通过重新调接口恢复。

**`deviceId` 的地位**：它是用户的隐式身份。必须跨 token 保留，否则每次 token 过期都会创建新 user，违反"持久匿名身份"设计。

### 6.2 应用启动流程

App 启动时（`main.ts` 挂载 App 前 或 进入任何业务路由前的路由守卫）必须执行：

1. 读 `localStorage.nc:deviceId`（可能为 null）
2. 调 `POST /session/init`，body 带 deviceId（null 也传）
3. 写回 `localStorage.nc:deviceId`（服务端可能回发新生成的值）
4. 写入 `localStorage.nc:token`
5. 更新 `useSessionStore`：userId、nickname、isNewUser
6. 放行路由

**错误处理：**
- `BANNED` → 跳 `BannedView`，不允许进入任何业务页
- `RATE_LIMITED_IP` → 停留在 `EntryView` 展示倒计时或"稍后再试"，不轮询
- 网络错误 → 展示重试按钮，手动触发

### 6.3 session store 必备字段

```ts
interface SessionState {
  deviceId: string | null
  token: string | null
  userId: number | null
  nickname: string | null
  isNewUser: boolean
  isBanned: boolean
}
```

`isBanned` 来自 401/403 拦截器标记，`BannedView` 监听此状态。

---

## 7. 消息收发（重要业务逻辑）

与后端 ACK 语义对齐（见后端 `CLAUDE.md` §6.5）。

### 7.1 发送消息的状态机

每条发出的消息在本地都有状态：

```
PENDING   → 客户端已入队，未发出或等待 ACK
SENT      → 已收到 ACK.status=OK，消息确认落在服务端
FAILED    → ACK.status=FAILED 或超时未收到 ACK
```

UI 应区分展示：

- PENDING：灰色 / loading 小图标
- SENT：正常显示，可选打勾
- FAILED：红色感叹号 + 点击可重试

### 7.2 clientMsgId 约定

- 发送前生成 UUID 作为 `clientMsgId`
- 本地消息以 `clientMsgId` 为键，收到 ACK 后用 `serverMsgId` 补全
- **去重**：收到广播的 `MSG_IN` 时，如果本地已有相同 `serverMsgId`（自己发的消息回环），跳过插入

### 7.3 列表渲染

- 使用虚拟滚动（消息多时）
- 滚动位置策略：新消息到达时若用户在底部则自动滚到底，否则只提示"有新消息"
- 消息时间戳客户端显示用 `ts`（服务端下发）

---

## 8. 开发规范

### 8.1 命名

| 类型 | 规则 | 示例 |
|---|---|---|
| 组件 | 大驼峰，多词 | `MessageList.vue`（不是 `List.vue`） |
| composable | `use` 前缀 + 小驼峰 | `useWebSocket` |
| store | 小驼峰，名词 | `useSessionStore` |
| 类型 / 接口 | 大驼峰 | `RoomInfo`、`SendMessageCommand` |
| 常量 | 全大写下划线 | `MAX_MESSAGE_LENGTH` |
| 文件 | 组件大驼峰；其他小驼峰 | |

### 8.2 TypeScript

- 严格模式（`strict: true`），禁用隐式 `any`
- 对外暴露的函数必须显式声明返回类型
- 避免 `as` 强转，除非能证明安全
- 用 discriminated union 表达有状态的类型（如消息状态）

### 8.3 组件

- Props 显式声明类型，避免运行时推导
- `<script setup lang="ts">` 作为默认写法
- 业务组件不直接调 `api` / 不直接操作 `WsClient`
- 纯展示组件（`components/`）尽量无状态，数据从 props 入
- 容器组件（`views/`）负责组合 store 和 composable

### 8.4 错误处理

- 网络错误：统一在 `http.ts` / `ws.ts` 拦截处理，业务层只处理"业务错误"
- 业务错误码对照表在 `src/constants/errorCodes.ts`，源头是 `API_SPEC.md` 的错误码章节
- UI 层展示用户友好提示，不直接把后端 message 原文甩给用户

### 8.5 测试

- `utils/` 必须有单元测试
- `composables/` 核心逻辑（ACK 跟踪、重连）必须有测试
- `stores/` action 测试
- `components/` 选择性测试，复杂交互必测

---

## 9. 与后端约定的前端应知事项

> 这些事项源自后端 `CLAUDE.md`，前端需感知的部分摘录在此。
> **若本节与 `CLAUDE.md` / `API_SPEC.md` 冲突，以后端文档为准。**

- **所有限流数值以 `docs/API_SPEC.md §4` 为权威**（消息长度、发送频率、昵称长度、IP 创建限额等）。前端做输入校验时读取 `API_SPEC.md` 的当前值，不在代码或文档中硬编码
- 超过发送频率会收到 `ACK.status=FAILED, code=RATE_LIMITED`，UI 需给出提示
- ACK 只有一次，语义是"消息已持久化不丢"；广播 `MSG_IN` 是独立事件
- 鉴权 JWT 通过 WebSocket 握手时走 query string（`ws://host/ws?token=xxx`），HTTP 走 `Authorization` header
- **匿名身份机制**：
  - 用户无需注册/登录；首次访问 `POST /session/init` 自动创建匿名身份
  - `deviceId` 由服务端首次生成并返回，前端**必须持久化到 localStorage**；回访时作为入参带上以获得稳定身份
  - `deviceId` 与 `token` 独立持久化：token 可过期重发，`deviceId` 永久保留（除用户清除浏览器数据）
  - 断线重连时 token 若失效，需重新走 `/session/init`（不需要任何用户交互）
- **昵称**：
  - 首次 session init 时服务端自动分配（`游客` + 6 位随机数字）
  - 用户可通过 `PATCH /user/nickname` 修改
  - 在房间内修改时若冲突会收到 `NICKNAME_IN_USE`，UI 必须提示用户换一个
  - **消息气泡的显示名必须用 `MSG_IN.nickname`**（后端下发的房内显示名），不要用 sessionStore 里的用户原昵称——二者在冲突场景下不同
- **封禁**：
  - 任何接口返回 `BANNED` 或 WS 收到 `KICK:BANNED` → 跳 `BannedView`，**不做任何自动重试、轮询、重连**
  - 不清除 `deviceId`（用户仍可能被解封），但显示封禁不可使用态
- **IP 限流**：
  - 启动时 `/session/init` 若返回 `RATE_LIMITED_IP`，在 `EntryView` 停留并提示稍后再试
  - 不自动轮询；用户点击重试按钮才再发一次

---

## 10. 文档维护（前端职责）

### 10.1 必须更新 `CLAUDE.md`（本文档）的场景

- 新增/修改目录结构
- 新增/修改分层约定
- 变更技术栈（例如确认了 UI 库、样式方案）
- 新增规范（命名、类型、测试）
- 用户提出新的前端约束

### 10.2 必须更新 `FEATURES.md` 的场景（仅前端条目）

- 前端职责功能的状态变化（PLANNED / IN_PROGRESS / DONE 等）
- 新增前端功能需求 → 新条目，ID 分配在 `F-08X` 段
- **不得**修改后端职责条目的状态

### 10.3 禁止修改 `API_SPEC.md`

如果发现 `API_SPEC.md` 与实际需求不匹配、有歧义、缺字段：
1. **停下来**，不要擅自决定
2. 向用户报告："`API_SPEC.md` §X 需要后端补充 / 澄清：<具体问题>"
3. 用户协调后端更新 `API_SPEC.md`
4. 基于新的 SPEC 继续开发

### 10.4 更新规则

与后端 `CLAUDE.md` §13.6 一致：
- 文档变更与代码变更同提交
- 更新后主动告知用户具体改了哪一节
- 模糊时先问再改
- `FEATURES.md` 变更日志必写

---

## 11. 分阶段开发计划

前端的主要开发集中在后端 Phase 7，但接口层可以和后端并行进行。建议阶段：

### Phase A：脚手架（可在后端 Phase 1 之后启动）
- Vite + Vue 3 + TS 初始化
- 目录结构搭好
- ESLint / Prettier / Vitest 配置
- 路由骨架（entry / chat / nickname / banned）

### Phase B：接口层（待后端完成 Phase 2~4 后）
- 根据 `API_SPEC.md` 生成类型
- HTTP 客户端 + 拦截器（含 401/403/429 处理）
- WebSocket 客户端封装
- mock 数据支持本地开发

### Phase C：匿名会话初始化（依赖后端 Phase 2）
- `useSessionInit` composable：启动读 localStorage → 调 `/session/init` → 持久化
- `useSessionStore`：deviceId / token / userId / nickname / isBanned
- 路由守卫：未初始化会话前不进入业务页
- `EntryView`：自动走会话初始化；异常时展示手动重试按钮
- `BannedView`：展示不可重试的封禁终止态
- `NicknameView`：修改昵称，处理 `NICKNAME_INVALID` / `NICKNAME_IN_USE`

### Phase D：聊天室核心（依赖后端 Phase 4~6）
- 定位获取与加入房间
- WS 连接建立（token 走 query）
- 消息列表 + 发送
- ACK 状态机（PENDING / SENT / FAILED）
- MSG_IN 显示用后端下发的 nickname（房内显示名）
- 断线重连（token 失效自动重走 session init）

### Phase E：打磨
- 错误提示（所有业务错误码对应 UI 文案）
- 空状态 / loading 状态
- 昵称修改入口
- 响应式布局
- 最小可用验证

每阶段完成后跑通、写测试、commit，再进下一步。
