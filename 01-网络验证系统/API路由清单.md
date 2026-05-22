---
文件位置: 01-网络验证系统/API路由清单.md
名称: API路由清单
作者: 蜂巢·大圣 (HiveGreatSage)
时间: 2026-05-22
版本: V1.2.6
状态: 草稿
关联文档:
  - "[[00-项目总控/项目总大纲]]"
  - "[[00-项目总控/术语表]]"
  - "[[00-项目总控/文档代码测试追踪矩阵]]"
  - "[[01-网络验证系统/README]]"
  - "[[01-网络验证系统/架构设计]]"
  - "[[01-网络验证系统/OpenAPI快照与接口契约治理规范]]"
  - "[[01-网络验证系统/接入契约]]"
  - "[[01-网络验证系统/API鉴权方案]]"
变更记录:
  - V1.2.6 (2026-05-22): 更新 OpenAPI 快照至 2026-05-21（142 paths / 121 schemas）
  - V1.2.5 (2026-05-18): 批次12收口；授权 PATCH 直改已下线，/users/:id 与 UserDetail 已下线，用户治理统一到 /users 工作台
  - V1.2.4 (2026-05-18): 批次11文档收口；已重新导出 OpenAPI 快照（112 paths / 97 schemas）并补充当前未收口差异（授权直改与用户详情双入口仍在）
  - V1.2.3 (2026-05-18): 用户授权升级接口新增 mode=topup_align（补时并批扣点）；升级预览/执行模式统一为 append / average / topup_align
  - V1.2.2 (2026-05-17): 用户列表筛选治理对齐源码；下线 creator_agent_tier_level / creator_agent_can_create_sub_agents / creator_agent_risk_status 三项筛选，仅保留 status / level / project_id / creator_agent_id
  - V1.2.1 (2026-05-12): 目录级治理第一批；修正 API 路由清单裸链并补充证据边界，明确手写清单不替代 OpenAPI 快照
  - V1.2.0 (2026-05-10): 对齐当前源码；修正用户管理路径为 /api/users/*；标记旧 balance / 旧硬删除接口已清理
  - V1.0.1: 同步 Verify 热更新链路小修复
  - V1.0.0: Obsidian 去漂移重构生成
---

# API路由清单

← 返回 [[00-项目总控/项目总大纲]] | 父节点: [[01-网络验证系统/架构设计]]

## 当前定位

本文件是手写 API 导航清单，不是唯一接口真相源。字段、请求体和响应体必须以 FastAPI OpenAPI 快照为准。

## 当前证据边界

- 本文件是手写 API 导航清单，只作为 D1 级文档索引。
- 字段、请求体、响应体、状态码必须以 FastAPI OpenAPI 快照和源码为准。
- 本轮已执行 `python scripts/export_openapi.py`，产物：`HiveGreatSage-Verify/docs/openapi/openapi_2026-05-21.json`、`openapi_routes_2026-05-21.md`（142 paths / 121 schemas）。
- 不得把本文中的路由存在直接写成"接口已联调通过"。

## 终端 User 接口

```text
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout
POST /api/auth/revoke-all
GET  /api/auth/me

POST /api/device/heartbeat
GET  /api/device/list
GET  /api/device/data

GET  /api/params/get
POST /api/params/set

GET  /api/update/check
GET  /api/update/download

GET  /api/client/network-config
```

说明：

```text
1. /api/device/list 与 /api/device/data 属于终端用户设备链。
2. 该链当前以 `device_id` 作为单设备查询参数。
3. 该链返回 `device_id`（设备编号）以及 `connection_type` / `connection_label`（连接标识）。
4. 后台设备监控不应复用该链路做原文展示。
```

## 管理员接口

```text
POST /admin/api/auth/login
POST /admin/api/session/logout
GET  /admin/api/dashboard
GET  /admin/api/login-logs/

/admin/api/projects/*
/admin/api/agents/{agent_id}/project-auths/*
/admin/api/devices/*
/admin/api/updates/*
/admin/api/accounting/*
/admin/api/prices/*
/admin/api/system-settings/*
/admin/api/project-access/*
/admin/api/agent-level-policies/*
/admin/api/audit-logs/*
/admin/api/agents/{agent_id}/business-profile
/admin/api/agents/{agent_id}/password
```

说明：

```text
1. /admin/api/dashboard 是当前管理后台 Dashboard 的真实数据源。
2. /admin/api/dashboard 当前返回工作台聚合字段，包括 today_accounting / level_distribution /
   expiring_auths / online_devices / active_projects_data / system_health。
3. /admin/api/devices/* 属于后台设备监控链，当前直接返回设备原文字段与连接标识字段。
4. 用户详情页设备绑定另走 /admin/api/users/{user_id}/devices，不与 /admin/api/devices/* 混为一条链。
5. /admin/api/updates/* 属于管理端热更新链路，当前按 `project_id + client_type` 组织。

   当前已确认管理端接口包括：
   - POST /admin/api/updates/{project_id}/{client_type}
   - GET  /admin/api/updates/{project_id}/{client_type}/latest
   - GET  /admin/api/updates/{project_id}/{client_type}/history

6. 当前前端管理页 UpdateManage.vue 与 UploadVersionForm.vue 已确认直接消费上述管理端链路：
   - latest：读取当前激活版本
   - history：读取版本历史
   - upload：上传并发布新版本包

7. /admin/api/updates/* 与客户端热更新链路职责不同；
   管理端负责发布、查看当前版本与历史，
   客户端链路仍走：
   - /api/update/check
   - /api/update/download
```

## 统计摘要接口

```text
GET /api/stats/platform
```

说明：

```text
1. /api/stats/platform 当前属于标准平台摘要接口，
   与 /admin/api/dashboard 的工作台聚合职责不同。

2. 当前已确认返回：
   - total_users
   - active_users
   - total_agents
   - total_projects
   - total_devices_bound
   - total_devices_online
   - total_authorizations
   - authorization_level_distribution

3. 当前 total_devices_online 已由 stats_service.py 直接汇总 Redis 在线设备统计，
   不再通过 stats.py 路由层补丁注入。
```

## Admin / Agent 共用用户接口

已确认：用户管理接口不是 `/admin/api/users/*`，当前源码注册路径为：

```text
/api/users/*
```

权限边界：

```text
Admin Token：可管理全部用户。
Agent Token：当前只管理自己创建的直属用户。
```

查询参数口径（`GET /api/users/`）：

```text
保留:
- page
- page_size
- status
- level
- project_id
- creator_agent_id

已下线（开发期不兼容）:
- creator_agent_tier_level
- creator_agent_can_create_sub_agents
- creator_agent_risk_status
```

升级设备数接口口径（`/api/users/{user_id}/authorizations/{auth_id}/upgrade*`）：

```text
mode 现行枚举:
- append       追加设备，沿用原到期口径
- average      新旧设备按剩余时长均摊
- topup_align  旧设备补时到新批次并批（补时部分扣点）
```

批次12收口结果（2026-05-18）：

```text
1. PATCH /api/users/{user_id}/authorizations/{auth_id}（授权字段直改）已下线，OpenAPI 不再暴露该方法。
2. /users/:id 路由与 UserDetail.vue 已下线，用户治理统一由 /users 页面“编辑抽屉”承载。
```

## 代理接口

```text
POST /api/agents/auth/login
POST /api/agents/session/logout
GET  /api/agents/me
GET  /api/agents/my/balance
GET  /api/agents/my/transactions
GET  /api/agents/my-projects
GET  /api/agents/scope/list
/api/agents/my/project-access/*
```

说明：

```text
1. /api/agents/me 当前用于“代理资料 + 轻量业务能力摘要”：
   - 基本账号信息
   - 父代理信息
   - 直属用户统计
   - 已授权项目
   - 业务画像能力字段（如 tier_level / can_create_sub_agents / can_auto_open_project /
     auto_open_project_limit / review_priority）

2. /api/agents/me 当前不返回钱包主数据。
   钱包余额与流水分别由：
   - /api/agents/my/balance
   - /api/agents/my/transactions
   提供。

3. /api/agents/me/dashboard 是代理端 Dashboard 聚合接口，
   当前主要服务 DashboardView.vue 的代理视角工作台。

   当前已确认返回的聚合块包括：
   - agent
   - wallet
   - users
   - online_devices
   - projects
   - expiring_auths
   - sub_agents
   - sub_expiring_auths

   该接口与 /api/agents/me 的职责不同；
   前者偏工作台聚合，后者偏资料与能力摘要。

4. /api/agents/scope/list 是代理端超级列表增强接口，
   不等同普通 /api/agents/ 列表。

   当前已确认在基础代理列表字段之外，额外补齐：
   - business_profile
   - balance
   - authorized_projects[].user_count

   其中：
   - business_profile 用于业务等级、风险状态、授信/下级代理能力展示
   - balance 用于列表中的可用点数与余额摘要展示
   - authorized_projects[].user_count 表示该代理直属授权用户数，不是全子树用户数
```

## 已清理的旧接口 / 旧文件

```text
app/routers/balance_admin.py
app/routers/balance_agent.py
scripts/setup_game_db.py
DELETE /admin/api/users/{user_id}
DELETE /admin/api/agents/{agent_id}
DELETE /admin/api/projects/{project_id}
```

说明：开发期不做旧接口兼容。旧硬删除接口已从主线清理；用户删除统一走 `/api/users/{user_id}` 软删除，代理/项目不再保留物理硬删除入口。

## 仍需运行验证

```text
OpenAPI 快照导出
前端菜单与路由联调
Admin / Agent / User 鉴权回归测试
热更新 check/download 端到端联调
```
