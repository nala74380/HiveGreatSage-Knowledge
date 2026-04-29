---
文件位置: 01-网络验证系统/API路由清单.md
名称: Verify API 路由清单
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-29
版本: V1.1.0
状态: 草稿
关联文档:
  - "[[项目总大纲]]"
  - "[[00-项目总控/项目总大纲_Verify状态同步补丁_2026-04-29]]"
  - "[[01-网络验证系统/架构设计]]"
  - "[[01-网络验证系统/当前实现反向沉淀_2026-04-29]]"
  - "[[01-网络验证系统/服务层业务规则_2026-04-29]]"
  - "[[01-网络验证系统/数据库模型清单_2026-04-29]]"
  - "[[01-网络验证系统/管理后台前端实现_2026-04-29]]"
  - "[[01-网络验证系统/测试基线审查_2026-04-29]]"
  - "[[编码规范_补充_Markdown文档规范]]"
变更记录:
  - V1.1.0: 基于 HiveGreatSage-Verify 当前源码反向整理 API 路由清单，按调用方、模块、鉴权身份、联调状态重排
  - V1.0.0: 初版 API 路由清单
---

# Verify API 路由清单

← 返回 [[项目总大纲]] | 父节点: [[01-网络验证系统/架构设计]]

## 一、文档目的

本文用于沉淀 `HiveGreatSage-Verify` 当前源码中的 API 路由清单。

本文目标：

```text
1. 明确 Verify 当前已经注册了哪些 API。
2. 明确每个 API 属于哪个 router。
3. 明确每个 API 的调用方：PCControl / AndroidScript / Admin Web / Agent Web。
4. 明确每个 API 的鉴权身份：User Token / Admin Token / Agent Token / 公开。
5. 明确每个 API 当前状态：源码已实现 / 待运行验证 / 待三端联调 / 待生产化。
6. 为 PC 中控、安卓脚本、管理后台、代理后台后续联调提供统一契约。
```

---

## 二、可信度说明

### 2.1 已确认

本文基于当前源码反向整理，已确认 `app/main.py` 中注册了以下 router：

```text
auth                     → /api/auth
users                    → /api/users
balance_agent            → /api/agents
project_access_agent     → /api/agents/my/project-access
agents                   → /api/agents
device                   → /api/device
params                   → /api/params
update                   → /api/update

admin                    → /admin/api
projects                 → /admin/api
update_admin             → /admin/api/updates
device_admin             → /admin/api/devices
stats                    → /api/stats
balance_admin            → /admin/api
project_access_admin     → /admin/api/project-access
agent_profile_admin      → /admin/api

health_check             → /health
```

### 2.2 待确认

本文是源码静态反向整理，不等于接口已经全部运行通过。

仍需确认：

```text
1. 当前环境 FastAPI 是否能完整启动。
2. /docs 是否能生成完整 OpenAPI。
3. 每个接口是否能通过真实 Token 调用。
4. Router 注册顺序是否仍存在动态路由抢占风险。
5. 前端页面调用路径是否与当前后端路由完全一致。
6. PCControl 调用路径是否与当前后端路由完全一致。
7. AndroidScript 调用路径是否与当前后端路由完全一致。
8. OpenAPI 导出的接口数量是否与本文手工反向清单一致。
```

### 2.3 状态标记

```text
✓ 源码已实现:
  当前源码中存在对应路由函数。

□ 待运行验证:
  需要通过 FastAPI / pytest / curl / httpx 当前环境重测。

□ 待三端联调:
  需要 PCControl / AndroidScript / Admin Web / Agent Web 调用验证。

□ 待生产化:
  需要权限、安全、压测、日志、监控、回滚等生产级验证。

⚠️ 待修订:
  路由命名、注册顺序、权限边界、前后端路径可能存在风险。
```

---

## 三、全局路由注册顺序

### 3.1 当前注册顺序

```python
app.include_router(auth.router, prefix="/api/auth", tags=["认证"])
app.include_router(users.router, prefix="/api/users", tags=["用户管理"])

app.include_router(balance_agent.router, prefix="/api/agents", tags=["代理余额"])

app.include_router(
    project_access_agent.router,
    prefix="/api/agents/my/project-access",
    tags=["代理项目准入"],
)

app.include_router(agents.router, prefix="/api/agents", tags=["代理管理"])
app.include_router(device.router, prefix="/api/device", tags=["设备数据"])
app.include_router(params.router, prefix="/api/params", tags=["脚本参数"])
app.include_router(update.router, prefix="/api/update", tags=["热更新"])

app.include_router(admin.router, prefix="/admin/api", tags=["管理后台"])
app.include_router(projects.router, prefix="/admin/api", tags=["项目管理"])
app.include_router(update_admin.router, prefix="/admin/api/updates", tags=["热更新管理"])
app.include_router(device_admin.router, prefix="/admin/api/devices", tags=["设备监控"])
app.include_router(stats.router, prefix="/api/stats", tags=["统计数据"])
app.include_router(balance_admin.router, prefix="/admin/api", tags=["点数管理"])

app.include_router(
    project_access_admin.router,
    prefix="/admin/api/project-access",
    tags=["项目准入管理"],
)

app.include_router(
    agent_profile_admin.router,
    prefix="/admin/api",
    tags=["代理业务管理"],
)
```

### 3.2 注册顺序关键风险

当前源码已特别标注：

```text
balance_agent 必须在 agents.router 之前注册。
```

原因：

```text
agents.router 中存在 /{agent_id} 动态路由。
如果 agents.router 先注册，则 /api/agents/catalog 可能被误匹配为 /api/agents/{agent_id}。
```

长期规则：

```text
1. 静态路径必须在参数化路径之前。
2. /api/agents/catalog 必须早于 /api/agents/{agent_id}。
3. /api/agents/my/... 必须早于 /api/agents/{agent_id}。
4. /api/users/creators/... 必须早于 /api/users/{user_id}。
5. 新增 router 时必须审查动态路径抢占。
```

---

## 四、调用方分组总览

### 4.1 PCControl 调用

PC 中控主要调用：

```text
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout
GET  /api/auth/me

GET  /api/device/list
GET  /api/device/data

POST /api/params/set

GET  /api/update/check
GET  /api/update/download
```

待确认：

```text
1. PCControl 是否也调用 GET /api/params/get 用于回显参数。
2. PCControl 是否需要调用 /api/stats/users/{user_id}/projects。
3. PCControl 是否需要独立 client_type=pc 的热更新通道。
```

### 4.2 AndroidScript 调用

安卓脚本主要调用：

```text
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout
GET  /api/auth/me

POST /api/device/heartbeat
POST /api/device/imsi

GET  /api/params/get

GET  /api/update/check
GET  /api/update/download
```

待确认：

```text
1. AndroidScript 是否已经实现 revoke-all 场景处理。
2. AndroidScript 是否已经实现 429 心跳限流处理。
3. AndroidScript 是否已经实现热更新 SHA-256 校验。
```

### 4.3 Admin Web 调用

管理员后台主要调用：

```text
POST   /admin/api/auth/login
GET    /admin/api/dashboard
GET    /admin/api/login-logs/

GET    /api/users/
POST   /api/users/
GET    /api/users/{user_id}
PATCH  /api/users/{user_id}
DELETE /api/users/{user_id}
PATCH  /api/users/{user_id}/password
POST   /api/users/{user_id}/authorizations
PATCH  /api/users/{user_id}/authorizations/{auth_id}
DELETE /api/users/{user_id}/authorizations/{auth_id}

GET    /api/agents/
POST   /api/agents/
GET    /api/agents/tree
GET    /api/agents/{agent_id}
PATCH  /api/agents/{agent_id}
GET    /api/agents/{agent_id}/subtree

GET    /admin/api/projects/
POST   /admin/api/projects/
GET    /admin/api/projects/{project_id}
PATCH  /admin/api/projects/{project_id}
DELETE /admin/api/projects/{project_id}

GET    /admin/api/updates/{project_id}/{client_type}/latest
GET    /admin/api/updates/{project_id}/{client_type}/history
POST   /admin/api/updates/{project_id}/{client_type}

GET    /admin/api/project-access/policies
PATCH  /admin/api/project-access/policies/{project_id}
GET    /admin/api/project-access/requests
POST   /admin/api/project-access/requests/{request_id}/approve
POST   /admin/api/project-access/requests/{request_id}/reject

GET    /admin/api/balance-transactions
GET    /admin/api/prices/{project_id}
PUT    /admin/api/prices/{project_id}/{user_level}
DELETE /admin/api/prices/{project_id}/{user_level}
GET    /admin/api/agents/{agent_id}/balance
POST   /admin/api/agents/{agent_id}/recharge
POST   /admin/api/agents/{agent_id}/credit
POST   /admin/api/agents/{agent_id}/freeze
POST   /admin/api/agents/{agent_id}/unfreeze
GET    /admin/api/agents/{agent_id}/transactions
GET    /admin/api/agents-full
```

### 4.4 Agent Web 调用

代理后台主要调用：

```text
POST /api/agents/auth/login
GET  /api/agents/me
GET  /api/agents/my-projects
GET  /api/agents/scope/list

GET  /api/agents/my/catalog
GET  /api/agents/catalog
GET  /api/agents/my/balance
GET  /api/agents/my/transactions

GET  /api/agents/my/project-access/catalog
POST /api/agents/my/project-access/requests
GET  /api/agents/my/project-access/requests
POST /api/agents/my/project-access/requests/{request_id}/cancel

GET  /api/users/
POST /api/users/
GET  /api/users/{user_id}
PATCH /api/users/{user_id}
DELETE /api/users/{user_id}
PATCH /api/users/{user_id}/password
POST /api/users/{user_id}/authorizations
PATCH /api/users/{user_id}/authorizations/{auth_id}
DELETE /api/users/{user_id}/authorizations/{auth_id}

GET  /api/stats/agents/my/summary
```

---

## 五、系统路由

### 5.1 健康检查

| 方法 | 路径 | Router | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/health` | `app/main.py` | Nginx / 部署监控 / 人工检查 | 公开 | ✓ 源码已实现，□ 待部署验证 |

返回示例：

```json
{
  "status": "ok",
  "environment": "development",
  "version": "0.1.0"
}
```

用途：

```text
1. Nginx upstream check。
2. 部署后快速检查服务是否存活。
3. 自动化运维探活。
```

待确认：

```text
1. 是否需要增加数据库连接检查。
2. 是否需要增加 Redis 连接检查。
3. 生产环境是否需要隐藏 environment。
```

---

## 六、认证路由 `/api/auth`

Router 文件：

```text
app/routers/auth.py
```

### 6.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/api/auth/login` | `login` | PCControl / AndroidScript | 公开 | ✓ 源码已实现，□ 待三端联调 |
| POST | `/api/auth/refresh` | `refresh` | PCControl / AndroidScript | Refresh Token | ✓ 源码已实现，□ 待运行验证 |
| POST | `/api/auth/logout` | `logout` | PCControl / AndroidScript | Access Token | ✓ 源码已实现，□ 待运行验证 |
| POST | `/api/auth/revoke-all` | `revoke_all` | PCControl / 安全操作 | Access Token | ✓ 源码已实现，⚠️ 即时失效能力待修订 |
| GET | `/api/auth/me` | `get_me` | PCControl / AndroidScript | Access Token | ✓ 源码已实现，□ 待三端联调 |

### 6.2 关键规则

```text
1. 登录接口有 D5 限流：同 IP 每分钟最多 10 次。
2. 登录成功后签发 Access Token 和 Refresh Token。
3. logout 会吊销当前 Access Token 并删除 Refresh Token。
4. revoke-all 会删除该用户所有 Refresh Token。
5. revoke-all 当前无法让所有已签发 Access Token 立即失效，只能等待 Access Token 自然过期或通过黑名单处理当前 Token。
```

### 6.3 风险点

```text
⚠️ revoke-all 的即时失效能力不足。
建议后续引入 user.token_version。
```

---

## 七、用户管理路由 `/api/users`

Router 文件：

```text
app/routers/users.py
```

### 7.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/api/users/` | `create_user_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待权限复测 |
| GET | `/api/users/` | `list_users_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待权限复测 |
| GET | `/api/users/creators/agents/{agent_id}` | `creator_agent_detail_endpoint` | Admin Web | Admin 优先 | ✓ 源码已实现，□ 待运行验证 |
| GET | `/api/users/{user_id}` | `get_user_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待权限复测 |
| PATCH | `/api/users/{user_id}` | `update_user_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待权限复测 |
| DELETE | `/api/users/{user_id}` | `delete_user_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待返点规则复测 |
| PATCH | `/api/users/{user_id}/password` | `update_user_password_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待运行验证 |
| POST | `/api/users/{user_id}/authorizations` | `grant_auth_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待扣点规则复测 |
| PATCH | `/api/users/{user_id}/authorizations/{auth_id}` | `update_auth_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待扣点规则复测 |
| DELETE | `/api/users/{user_id}/authorizations/{auth_id}` | `revoke_auth_endpoint` | Admin Web / Agent Web | Admin 或 Agent | ✓ 源码已实现，□ 待返点规则复测 |

### 7.2 当前核心口径

```text
1. User 是账号主体。
2. Authorization 是用户在某项目下的授权记录。
3. 项目内等级、设备数、到期时间全部归属 Authorization。
4. Admin Token 可操作所有用户。
5. Agent Token 只能操作自己权限范围内的用户。
```

### 7.3 查询参数

`GET /api/users/` 支持：

```text
page
page_size
status
level
project_id
creator_agent_id
```

`GET /api/users/{user_id}` 支持：

```text
project_id
```

### 7.4 风险点

```text
1. Admin / Agent 共用同一路由，需要严格测试权限边界。
2. 删除用户涉及返点规则，不能只测试状态变化。
3. 授权变更涉及扣点、到期时间、设备数，必须做账务一致性测试。
4. /creators/agents/{agent_id} 必须在 /{user_id} 之前注册。
```

---

## 八、代理管理路由 `/api/agents`

Router 文件：

```text
app/routers/agents.py
```

### 8.1 代理身份与管理路由

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/api/agents/auth/login` | `agent_login_endpoint` | Agent Web | 公开 | ✓ 源码已实现，□ 待前端联调 |
| GET | `/api/agents/me` | `get_my_profile` | Agent Web | Agent Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/api/agents/my-projects` | `get_my_authorized_projects` | Agent Web | Agent Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/api/agents/scope/list` | `list_agents_in_scope_endpoint` | Agent Web | Agent Token | ✓ 源码已实现，□ 待权限复测 |
| POST | `/api/agents/` | `create_agent_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| GET | `/api/agents/` | `list_agents_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| GET | `/api/agents/tree` | `get_full_agent_tree` | Admin Web | Admin Token | ✓ 源码已实现，□ 待递归边界复测 |
| GET | `/api/agents/{agent_id}` | `get_agent_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| GET | `/api/agents/{agent_id}/subtree` | `get_agent_subtree_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待递归边界复测 |
| PATCH | `/api/agents/{agent_id}` | `update_agent_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |

### 8.2 注册顺序规则

必须保证以下静态路由早于 `/{agent_id}`：

```text
/api/agents/auth/login
/api/agents/me
/api/agents/my-projects
/api/agents/scope/list
/api/agents/tree
/api/agents/my/catalog
/api/agents/catalog
/api/agents/my/balance
/api/agents/my/transactions
/api/agents/my/project-access/...
```

### 8.3 风险点

```text
1. 多级代理递归查询需要测试性能。
2. Agent Token 权限范围必须避免越权。
3. 动态路径抢占是此模块长期风险。
4. Agent 与 Admin 共用部分用户管理接口，权限判断必须统一。
```

---

## 九、代理余额与定价路由 `/api/agents`

Router 文件：

```text
app/routers/balance_agent.py
```

挂载前缀：

```text
/api/agents
```

### 9.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/api/agents/my/catalog` | `my_price_catalog` | Agent Web | Agent Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/api/agents/catalog` | `price_catalog_compat` | Agent Web | Agent Token | ✓ 源码已实现，⚠️ 兼容旧路径 |
| GET | `/api/agents/my/balance` | `my_balance` | Agent Web | Agent Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/api/agents/my/transactions` | `my_transactions` | Agent Web | Agent Token | ✓ 源码已实现，□ 待流水测试 |

### 9.2 查询参数

`GET /api/agents/my/transactions` 支持：

```text
page
page_size
tx_type
related_user_id
related_project_id
```

### 9.3 风险点

```text
1. /api/agents/catalog 是兼容路径，长期建议前端统一使用 /api/agents/my/catalog。
2. 点数流水属于财务敏感数据，必须单独测试。
3. 代理只能查看自己的余额和流水，不能越权查看上级或兄弟代理。
```

---

## 十、代理项目准入路由 `/api/agents/my/project-access`

Router 文件：

```text
app/routers/project_access_agent.py
```

### 10.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/api/agents/my/project-access/catalog` | `my_project_catalog` | Agent Web | Agent Token | ✓ 源码已实现，□ 待前端联调 |
| POST | `/api/agents/my/project-access/requests` | `create_project_auth_request` | Agent Web | Agent Token | ✓ 源码已实现，□ 待准入策略测试 |
| GET | `/api/agents/my/project-access/requests` | `my_project_auth_requests` | Agent Web | Agent Token | ✓ 源码已实现，□ 待前端联调 |
| POST | `/api/agents/my/project-access/requests/{request_id}/cancel` | `cancel_project_auth_request` | Agent Web | Agent Token | ✓ 源码已实现，□ 待运行验证 |

### 10.2 业务含义

```text
1. catalog 返回代理可申请或可售卖的项目目录。
2. requests 用于代理提交项目开通申请。
3. cancel 用于代理取消自己的待审核申请。
4. 满足条件时可能自动开通。
```

### 10.3 风险点

```text
1. 代理不能取消别人的申请。
2. 已审核申请是否允许取消需要业务规则确认。
3. 自动开通条件必须有单独测试。
4. 上级代理准入是否继承到下级代理，需补决策。
```

---

## 十一、设备数据路由 `/api/device`

Router 文件：

```text
app/routers/device.py
```

### 11.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/api/device/heartbeat` | `heartbeat` | AndroidScript | User Token | ✓ 源码已实现，□ 待 Redis/Celery 联调 |
| GET | `/api/device/list` | `list_devices` | PCControl | User Token | ✓ 源码已实现，□ 待 PC 联调 |
| POST | `/api/device/imsi` | `upload_device_imsi` | AndroidScript | User Token | ✓ 源码已实现，⚠️ 合规与用途待确认 |
| GET | `/api/device/data` | `device_data` | PCControl | User Token | ✓ 源码已实现，□ 待 PC 联调 |

### 11.2 关键规则

```text
1. heartbeat 使用 D5 限流：同 user_id 每分钟 ≤ 4 次。
2. heartbeat 写入 Redis 缓冲层后立即返回，不等待数据库落库。
3. list 供 PC 中控拉取当前用户在当前游戏下的所有设备状态。
4. list 在线设备来自 Redis，离线设备来自游戏库历史落库数据。
5. data 优先读 Redis，Redis 无数据时回落到游戏库。
6. imsi 不参与登录验证，仅作为辅助标识存储。
```

### 11.3 查询参数

`GET /api/device/data`：

```text
device_fingerprint: string
```

### 11.4 风险点

```text
1. 设备绑定维度必须最终统一为 user_id + game_project_id + device_id/device_fingerprint。
2. IMSI 涉及合规与隐私，必须明确用途、权限、脱敏和保留周期。
3. heartbeat 限流应与 AndroidScript 实际上报间隔一致。
4. PCControl 轮询频率建议与 D2 决策一致，避免压力过大。
```

---

## 十二、脚本参数路由 `/api/params`

Router 文件：

```text
app/routers/params.py
```

### 12.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/api/params/get` | `get_params` | AndroidScript / PCControl 可选 | User Token | ✓ 源码已实现，□ 待安卓联调 |
| POST | `/api/params/set` | `set_params` | PCControl | User Token | ✓ 源码已实现，□ 待 PC 联调 |

### 12.2 关键规则

```text
1. GET 用于安卓脚本启动时拉取当前用户在本游戏下的全部脚本参数。
2. POST 用于 PC 中控保存用户修改后的参数值。
3. 参数数据来自游戏库。
4. 按 game_project_code 动态获取游戏库 Session。
5. set 支持部分成功：每一项独立验证，某项失败不影响其他项写入。
```

### 12.3 支持参数值格式

```text
int:    "42"
float:  "3.14"
bool:   "true" 或 "false"
string: "北境"
enum:   "1"
```

### 12.4 风险点

```text
1. 参数定义与参数值必须区分。
2. PCControl 前端展示必须与参数类型一致。
3. AndroidScript 需要处理缺省值与空值。
4. 参数变更是否需要版本号或变更日志，待确认。
```

---

## 十三、客户端热更新路由 `/api/update`

Router 文件：

```text
app/routers/update.py
```

### 13.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/api/update/check` | `check_update_endpoint` | PCControl / AndroidScript | User Token | ✓ 源码已实现，□ 待三端联调 |
| GET | `/api/update/download` | `download_update_endpoint` | PCControl / AndroidScript | User Token | ✓ 源码已实现，□ 待下载验证 |

### 13.2 查询参数

`GET /api/update/check`：

```text
client_type: pc | android
current_version: MAJOR.MINOR.PATCH
```

`GET /api/update/download`：

```text
client_type: pc | android
```

### 13.3 关键规则

```text
1. client_type 仅允许 pc 或 android。
2. current_version 格式必须为 x.y.z，例如 1.0.0。
3. check 返回是否需要更新。
4. download 返回签名限时下载 URL。
5. 下载完成后客户端必须校验 checksum_sha256。
```

### 13.4 风险点

```text
1. PCControl 更新安装流程未验证。
2. AndroidScript installLrPkg 流程未验证。
3. 签名 URL 过期重试需要客户端支持。
4. 强制更新 force_update 需要客户端阻断运行逻辑。
```

---

## 十四、管理后台基础路由 `/admin/api`

Router 文件：

```text
app/routers/admin.py
```

### 14.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/admin/api/auth/login` | `admin_login_endpoint` | Admin Web | 公开 | ✓ 源码已实现，□ 待前端联调 |
| GET | `/admin/api/dashboard` | `admin_dashboard` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/admin/api/login-logs/` | `list_login_logs` | Admin Web | Admin Token | ✓ 源码已实现，□ 待分页过滤测试 |
| GET | `/admin/api/debug/device-bindings/{user_id}` | `debug_device_bindings` | Admin Web / 调试 | Admin Token | ✓ 源码已实现，⚠️ 生产环境建议限制 |
| GET | `/admin/api/users/{user_id}/devices` | `get_user_devices` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| DELETE | `/admin/api/users/{user_id}/devices/{binding_id}` | `unbind_device` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| DELETE | `/admin/api/users/{user_id}` | `delete_user` | Admin Web | Admin Token | ✓ 源码已实现，⚠️ 与 /api/users/{user_id} 存在职责重叠 |
| DELETE | `/admin/api/agents/{agent_id}` | `delete_agent` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| DELETE | `/admin/api/projects/{project_id}` | `delete_project` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |

### 14.2 查询参数

`GET /admin/api/login-logs/` 支持：

```text
page
page_size
success
client_type
date_from
date_to
```

### 14.3 风险点

```text
1. debug/device-bindings 属于调试端点，生产环境建议下线或增加更严格权限。
2. DELETE /admin/api/users/{user_id} 与 DELETE /api/users/{user_id} 都存在用户删除能力，职责需要统一。
3. 后台设备解绑需要审计日志。
```

---

## 十五、项目管理路由 `/admin/api`

Router 文件：

```text
app/routers/projects.py
```

### 15.1 项目 CRUD

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/admin/api/projects/` | `create_project_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/admin/api/projects/` | `list_projects_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/admin/api/projects/{project_id}` | `get_project_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| PATCH | `/admin/api/projects/{project_id}` | `update_project_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |

说明：

```text
创建游戏项目时会自动生成 db_name。
验证项目无独立数据库。
```

### 15.2 管理员给代理授权项目

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/admin/api/agents/{agent_id}/project-auths/` | `grant_agent_auth_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| GET | `/admin/api/agents/{agent_id}/project-auths/` | `list_agent_auths_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| PATCH | `/admin/api/agents/{agent_id}/project-auths/{auth_id}` | `update_agent_auth_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |
| DELETE | `/admin/api/agents/{agent_id}/project-auths/{auth_id}` | `revoke_agent_auth_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待运行验证 |

### 15.3 风险点

```text
1. 项目 CRUD 与项目准入策略必须联动。
2. 验证项目与游戏项目的 db_name 生成规则不同，必须测试。
3. 代理项目授权与项目准入体系存在交叉，需确认是否长期并存。
```

---

## 十六、热更新管理路由 `/admin/api/updates`

Router 文件：

```text
app/routers/update_admin.py
```

### 16.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| POST | `/admin/api/updates/{project_id}/{client_type}` | `upload_version_endpoint` | Admin Web | Admin Token | ✓ 源码已实现，□ 待端到端验证 |
| GET | `/admin/api/updates/{project_id}/{client_type}/latest` | `get_latest_version` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| GET | `/admin/api/updates/{project_id}/{client_type}/history` | `get_version_history` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |

### 16.2 当前 V2 口径

```text
1. 版本记录从游戏库迁移到主库 version_record 表。
2. 支持游戏项目和普通验证项目。
3. 不再需要连接游戏库。
4. 上传成功后会清理 Redis 版本缓存。
```

### 16.3 上传参数

`POST /admin/api/updates/{project_id}/{client_type}` 使用 `multipart/form-data`：

```text
project_id: path int
client_type: path pc | android
version: form x.y.z
force_update: form bool
release_notes: form string | null
file: UploadFile
```

### 16.4 文件限制

```text
允许扩展名:
  .lrj
  .zip
  .apk
  .exe

最大文件大小:
  500 MB
```

### 16.5 风险点

```text
1. 热更新上传是高风险接口，必须记录操作审计。
2. .exe / .apk / .lrj 更新流程必须分别实机验证。
3. 重复上传同版本会更新已有记录，必须确认是否符合发布治理。
4. Redis 缓存 key 当前为 update:latest:{project.code_name}:{client_type}，需要写入 Redis Key 规范。
```

---

## 十七、设备监控路由 `/admin/api/devices`

Router 文件：

```text
app/routers/device_admin.py
```

### 17.1 当前已知状态

已确认该 router 已在 `app/main.py` 中注册：

```text
prefix="/admin/api/devices"
tags=["设备监控"]
```

### 17.2 待反向展开

待从 `app/routers/device_admin.py` 逐行补齐：

```text
GET /admin/api/devices/
GET /admin/api/devices/{device_id}
GET /admin/api/devices/{device_id}/runtime
GET /admin/api/devices/{device_id}/logs
```

### 17.3 风险点

```text
1. Admin 与 Agent 是否共用设备监控视图，需确认权限边界。
2. 设备日志是否已经真实落库，需确认。
3. 设备 runtime 数据来源 Redis 还是数据库，需确认。
```

> 注：本小节先保留为待展开项，避免把未逐行确认的路由细节写成已确认事实。

---

## 十八、统计路由 `/api/stats`

Router 文件：

```text
app/routers/stats.py
```

挂载前缀：

```text
/api/stats
```

### 18.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/api/stats/users/{user_id}/projects` | `user_project_stats` | Admin Web | Admin Token | ✓ 源码已实现，□ 待统计口径验证 |
| GET | `/api/stats/agents/my/summary` | `agent_project_summary` | Agent Web | Agent Token | ✓ 源码已实现，□ 待代理前端联调 |
| GET | `/api/stats/platform` | `platform_summary` | Admin Web | Admin Token | ✓ 源码已实现，□ 待统计口径验证 |

### 18.2 风险点

```text
1. /api/stats/platform 虽然挂在 /api/stats，但实际是 Admin Token 使用。
2. Redis 在线设备数统计使用 device:runtime:* key，需要与 Redis Key 规范统一。
3. 统计口径必须区分：绑定设备、激活设备、在线设备、历史设备。
```

---

## 十九、点数管理路由 `/admin/api`

Router 文件：

```text
app/routers/balance_admin.py
```

### 19.1 管理员全局流水

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/admin/api/balance-transactions` | `global_balance_transactions` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |

查询参数：

```text
page
page_size
tx_type
agent_id
related_user_id
related_project_id
```

### 19.2 项目定价

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/admin/api/prices/{project_id}` | `get_prices` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| PUT | `/admin/api/prices/{project_id}/{user_level}` | `set_price` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |
| DELETE | `/admin/api/prices/{project_id}/{user_level}` | `delete_price` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |

### 19.3 代理余额管理

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/admin/api/agents/{agent_id}/balance` | `get_balance` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| POST | `/admin/api/agents/{agent_id}/recharge` | `recharge` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |
| POST | `/admin/api/agents/{agent_id}/credit` | `credit` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |
| POST | `/admin/api/agents/{agent_id}/freeze` | `freeze` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |
| POST | `/admin/api/agents/{agent_id}/unfreeze` | `unfreeze` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |
| GET | `/admin/api/agents/{agent_id}/transactions` | `transactions` | Admin Web | Admin Token | ✓ 源码已实现，□ 待财务测试 |
| GET | `/admin/api/agents-full` | `agents_full_list` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |

### 19.4 风险点

```text
1. 点数相关接口全部属于财务敏感接口。
2. recharge / credit / freeze / unfreeze 必须写审计日志。
3. set_price 影响后续授权扣费，必须严格记录变更。
4. 返点、扣点、流水必须单独建立测试用例。
```

---

## 二十、项目准入管理路由 `/admin/api/project-access`

Router 文件：

```text
app/routers/project_access_admin.py
```

### 20.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/admin/api/project-access/policies` | `project_access_policies` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| PATCH | `/admin/api/project-access/policies/{project_id}` | `update_policy` | Admin Web | Admin Token | ✓ 源码已实现，□ 待准入规则测试 |
| GET | `/admin/api/project-access/requests` | `project_auth_requests` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| POST | `/admin/api/project-access/requests/{request_id}/approve` | `approve_request` | Admin Web | Admin Token | ✓ 源码已实现，□ 待准入规则测试 |
| POST | `/admin/api/project-access/requests/{request_id}/reject` | `reject_request` | Admin Web | Admin Token | ✓ 源码已实现，□ 待准入规则测试 |

### 20.2 查询参数

`GET /admin/api/project-access/policies`：

```text
page
page_size
```

`GET /admin/api/project-access/requests`：

```text
page
page_size
status
agent_id
project_id
```

### 20.3 风险点

```text
1. approve 可能触发代理项目准入开通，必须记录审核管理员。
2. reject 需要记录原因。
3. 准入策略变更后，已有代理准入是否受影响，需要补业务决策。
```

---

## 二十一、代理业务管理路由 `/admin/api`

Router 文件：

```text
app/routers/agent_profile_admin.py
```

### 21.1 路由清单

| 方法 | 路径 | 函数 | 调用方 | 鉴权 | 状态 |
|---|---|---|---|---|---|
| GET | `/admin/api/agent-level-policies` | `agent_level_policies` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| PATCH | `/admin/api/agent-level-policies/{level}` | `update_level_policy` | Admin Web | Admin Token | ✓ 源码已实现，□ 待业务规则复核 |
| GET | `/admin/api/agents/{agent_id}/business-profile` | `agent_business_profile` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| PATCH | `/admin/api/agents/{agent_id}/business-profile` | `update_business_profile` | Admin Web | Admin Token | ✓ 源码已实现，□ 待前端联调 |
| POST | `/admin/api/agents/{agent_id}/password` | `reset_password` | Admin Web | Admin Token | ✓ 源码已实现，□ 待安全审查 |

### 21.2 风险点

```text
1. 重置代理密码必须有审计日志。
2. 等级策略变更可能影响代理权益，需要记录变更历史。
3. business-profile 是业务画像，不应和核心鉴权字段混淆。
```

---

## 二十二、接口权限矩阵

### 22.1 User Token

User Token 可访问：

```text
/api/auth/logout
/api/auth/revoke-all
/api/auth/me
/api/device/heartbeat
/api/device/list
/api/device/imsi
/api/device/data
/api/params/get
/api/params/set
/api/update/check
/api/update/download
```

### 22.2 Admin Token

Admin Token 可访问：

```text
/admin/api/**
/api/users/**
/api/agents/**
/api/stats/users/{user_id}/projects
/api/stats/platform
```

注意：

```text
/api/users/** 当前也允许 Agent Token，但权限范围不同。
```

### 22.3 Agent Token

Agent Token 可访问：

```text
/api/agents/auth/login  # 登录本身公开
/api/agents/me
/api/agents/my-projects
/api/agents/scope/list
/api/agents/my/catalog
/api/agents/catalog
/api/agents/my/balance
/api/agents/my/transactions
/api/agents/my/project-access/**
/api/users/**
/api/stats/agents/my/summary
```

注意：

```text
Agent Token 访问 /api/users/** 时必须限制到自己创建或权限范围内的用户。
```

### 22.4 公开接口

公开接口：

```text
GET  /health
POST /api/auth/login
POST /api/auth/refresh       # 使用 Refresh Token，不需要 Access Token
POST /admin/api/auth/login
POST /api/agents/auth/login
```

---

## 二十三、前后端联调优先级

### 23.1 P0：三端核心闭环

```text
1. POST /api/auth/login
2. GET  /api/auth/me
3. POST /api/device/heartbeat
4. GET  /api/device/list
5. GET  /api/device/data
6. GET  /api/params/get
7. POST /api/params/set
8. GET  /api/update/check
9. GET  /api/update/download
```

完成标准：

```text
AndroidScript 能登录、上报心跳、拉参数、检查更新。
PCControl 能登录、拉设备、看数据、改参数、检查更新。
```

### 23.2 P1：管理后台基础功能

```text
1. POST /admin/api/auth/login
2. GET  /admin/api/dashboard
3. GET  /api/users/
4. POST /api/users/
5. GET  /api/agents/
6. POST /api/agents/
7. GET  /admin/api/projects/
8. POST /admin/api/projects/
9. GET  /admin/api/login-logs/
```

完成标准：

```text
管理员能创建项目、代理、用户，能查看平台基础状态。
```

### 23.3 P1：代理后台基础功能

```text
1. POST /api/agents/auth/login
2. GET  /api/agents/me
3. GET  /api/agents/my/balance
4. GET  /api/agents/my/transactions
5. GET  /api/agents/my/catalog
6. GET  /api/agents/my/project-access/catalog
7. POST /api/agents/my/project-access/requests
```

完成标准：

```text
代理能登录、查看余额、查看项目目录、提交项目申请。
```

### 23.4 P2：点数、准入、热更新

```text
1. GET  /admin/api/prices/{project_id}
2. PUT  /admin/api/prices/{project_id}/{user_level}
3. POST /admin/api/agents/{agent_id}/recharge
4. POST /admin/api/agents/{agent_id}/credit
5. GET  /admin/api/balance-transactions
6. GET  /admin/api/project-access/policies
7. PATCH /admin/api/project-access/policies/{project_id}
8. POST /admin/api/project-access/requests/{request_id}/approve
9. POST /admin/api/updates/{project_id}/{client_type}
10. GET /admin/api/updates/{project_id}/{client_type}/latest
11. GET /admin/api/updates/{project_id}/{client_type}/history
```

完成标准：

```text
点数、项目准入、热更新三个平台能力完成后台联调。
```

---

## 二十四、当前 API 风险清单

### R001 动态路由抢占

风险：

```text
/api/agents/catalog
/api/agents/my/...
/api/agents/tree
```

可能被：

```text
/api/agents/{agent_id}
```

抢占。

当前处理：

```text
balance_agent 已放在 agents.router 前。
agents.py 内部也强调静态路径必须放在动态路径之前。
```

后续要求：

```text
新增 /api/agents 下接口时必须先审查注册顺序。
```

### R002 Admin / Agent 共用 `/api/users/**`

风险：

```text
同一套用户管理接口同时允许 Admin Token 与 Agent Token。
```

好处：

```text
减少重复接口。
```

风险：

```text
权限判断一旦遗漏，代理可能越权操作非自己范围用户。
```

后续要求：

```text
1. 为 /api/users/** 写 Admin / Agent 双身份权限测试。
2. 明确 Agent 可操作范围。
3. 所有用户删除、授权、改密均需审计。
```

### R003 `/admin/api/users/{user_id}` 与 `/api/users/{user_id}` 存在重叠

风险：

```text
两个路径都能删除用户。
```

后续建议：

```text
1. 保留 /api/users/{user_id} 作为统一用户删除接口。
2. 将 /admin/api/users/{user_id} 标记为兼容或调试接口。
3. 后续统一前端只调用一种路径。
```

### R004 调试接口生产暴露

风险接口：

```text
/admin/api/debug/device-bindings/{user_id}
```

后续建议：

```text
1. 生产环境禁用。
2. 或增加 SUPER_ADMIN 权限。
3. 或移入 /admin/api/debug 并通过 settings.DEBUG 控制。
```

### R005 热更新管理路由与旧文档不一致

当前源码 V2 热更新管理路由是：

```text
POST /admin/api/updates/{project_id}/{client_type}
GET  /admin/api/updates/{project_id}/{client_type}/latest
GET  /admin/api/updates/{project_id}/{client_type}/history
```

旧文档中可能仍存在：

```text
POST /admin/api/updates/
GET  /admin/api/updates/
GET  /admin/api/updates/{version_id}
POST /admin/api/updates/{version_id}/activate
```

后续要求：

```text
1. 以当前源码 V2 路由为准。
2. 删除或标记旧文档中的旧路由。
3. 前端热更新页面必须同步新路径。
```

### R006 统计接口挂载路径与身份不完全一致

当前：

```text
/admin api 路由不是全部在 /admin/api 下。
/api/stats/platform 实际需要 Admin Token。
```

后续建议：

```text
1. 接受当前路径，但在文档中明确鉴权身份。
2. 或后续新增 /admin/api/stats/platform 兼容路径。
```

### R007 IMSI 上传合规风险

接口：

```text
POST /api/device/imsi
```

风险：

```text
IMSI 属于敏感设备标识。
```

后续要求：

```text
1. 明确是否必须采集。
2. 明确用途。
3. 明确脱敏展示。
4. 明确保留周期。
5. 明确代理/管理员是否可见。
```

---

## 二十五、建议生成 OpenAPI 快照

建议在 Verify 当前环境可启动后执行：

```bash
python - <<'PY'
import json
from app.main import app

schema = app.openapi()
with open("docs/openapi_2026-04-29.json", "w", encoding="utf-8") as f:
    json.dump(schema, f, ensure_ascii=False, indent=2)

print("OpenAPI 路由数量:", len(schema.get("paths", {})))
PY
```

然后将结果沉淀为：

```text
01-网络验证系统/OpenAPI快照_2026-04-29.md
```

用途：

```text
1. 与本文人工路由清单交叉验证。
2. 给 PCControl 生成 API Client。
3. 给 AndroidScript 生成接口契约。
4. 给前端检查 Axios 调用路径。
```

---

## 二十六、推荐下一步

### 26.1 立即执行

```text
1. 把本文复制进 Obsidian:
   01-网络验证系统/API路由清单.md

2. 对照当前 frontend/src/api 目录，生成:
   01-网络验证系统/前后端接口对照表.md

3. 对照 PCControl core/api_client，生成:
   02-PC中控框架/Verify接口调用清单.md

4. 对照 AndroidScript framework/verify.lua、heartbeat.lua、cloud_sync.lua、updater.lua，生成:
   03-安卓脚本框架/Verify接口调用清单.md
```

### 26.2 当前最优测试顺序

```text
Step 1:
  GET /health

Step 2:
  POST /admin/api/auth/login

Step 3:
  POST /api/agents/auth/login

Step 4:
  POST /api/auth/login

Step 5:
  POST /api/device/heartbeat

Step 6:
  GET /api/device/list

Step 7:
  GET /api/params/get

Step 8:
  POST /api/params/set

Step 9:
  GET /api/update/check

Step 10:
  POST /admin/api/updates/{project_id}/{client_type}
```

---

## 二十七、结论

当前 Verify API 已经覆盖平台 MVP+ 所需的主要接口：

```text
1. 用户登录认证。
2. Refresh Token。
3. 登出与踢出所有设备。
4. 用户管理。
5. 用户项目授权。
6. 代理管理。
7. 代理余额与流水。
8. 代理项目准入。
9. 设备心跳。
10. 设备列表与运行数据。
11. IMSI 上传。
12. 脚本参数拉取与保存。
13. 客户端热更新检查与下载。
14. 管理员后台登录与仪表盘。
15. 项目管理。
16. 热更新包上传与历史查询。
17. 点数充值、授信、冻结、定价。
18. 项目准入审核。
19. 代理业务画像与等级策略。
20. 统计接口。
```

当前最准确状态：

```text
API 源码已实现较完整；
但必须通过 OpenAPI 快照、pytest、前端联调、PCControl 联调、AndroidScript 实机联调后，才能标记为“接口契约稳定”。
```