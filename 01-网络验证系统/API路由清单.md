---
文件位置: 01-网络验证系统/API路由清单_2026-04-29.md
名称: Verify API 路由清单
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-29
版本: V1.0.0
状态: 草稿
关联文档:
  - "[[项目总大纲]]"
  - "[[01-网络验证系统/当前实现反向沉淀_2026-04-29]]"
  - "[[编码规范_补充_Markdown文档规范]]"
变更记录:
  - V1.0.0: 从 app/main.py 路由注册与 app/routers/*.py 装饰器反向生成当前 API 清单
---

# Verify API 路由清单

← 返回 [[01-网络验证系统/当前实现反向沉淀_2026-04-29]] | 父节点: [[01-网络验证系统/架构设计]]

## 一、阅读结论

已确认：当前 FastAPI 后端共静态识别到 **87 个路由**。这些路由来自 `app/main.py` 中的 `include_router(...)` 注册与 `app/routers/*.py` 中的 `@router.*` 装饰器。

待确认：本文是静态路由清单，不代表每个接口已经在当前环境运行通过；运行状态以重新执行 pytest、前端联调、PC/安卓端调用结果为准。

## 认证（auth.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `POST` | `/api/auth/login` | `login` | `LoginResponse` | `db, redis` |  |
| `POST` | `/api/auth/logout` | `logout` | `MessageResponse` | `credentials, redis` |  |
| `GET` | `/api/auth/me` | `get_me` | `MeResponse` | `credentials, current_user` |  |
| `POST` | `/api/auth/refresh` | `refresh` | `TokenResponse` | `db, redis` |  |
| `POST` | `/api/auth/revoke-all` | `revoke_all` | `MessageResponse` | `credentials, redis` | 踢出所有设备 |

## 用户管理（users.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/users/` | `list_users_endpoint` | `UserListResponse` | `caller, db` |  |
| `POST` | `/api/users/` | `create_user_endpoint` | `UserResponse` | `caller, db` |  |
| `GET` | `/api/users/creators/agents/{agent_id}` | `creator_agent_detail_endpoint` | `CreatorAgentDetailResponse` | `caller, db` |  |
| `DELETE` | `/api/users/{user_id}` | `delete_user_endpoint` | `` | `caller, db` |  |
| `GET` | `/api/users/{user_id}` | `get_user_endpoint` | `UserResponse` | `caller, db` |  |
| `PATCH` | `/api/users/{user_id}` | `update_user_endpoint` | `UserResponse` | `caller, db` |  |
| `POST` | `/api/users/{user_id}/authorizations` | `grant_auth_endpoint` | `AuthorizationResponse` | `caller, db` |  |
| `DELETE` | `/api/users/{user_id}/authorizations/{auth_id}` | `revoke_auth_endpoint` | `` | `caller, db` |  |
| `PATCH` | `/api/users/{user_id}/authorizations/{auth_id}` | `update_auth_endpoint` | `AuthorizationResponse` | `caller, db` |  |
| `PATCH` | `/api/users/{user_id}/password` | `update_user_password_endpoint` | `UserPasswordUpdateResponse` | `caller, db` |  |

## 代理余额（balance_agent.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/agents/catalog` | `price_catalog_compat` | `` | `current_agent, db` | 全项目定价目录（代理查看，兼容旧路径） |
| `GET` | `/api/agents/my/balance` | `my_balance` | `` | `current_agent, db` | 代理查询自己余额 |
| `GET` | `/api/agents/my/catalog` | `my_price_catalog` | `` | `current_agent, db` | 全项目定价目录（代理查看，推荐路径） |
| `GET` | `/api/agents/my/transactions` | `my_transactions` | `` | `current_agent, db` | 代理查询自己流水 |

## 代理项目准入（project_access_agent.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/agents/my/project-access/catalog` | `my_project_catalog` | `list[AgentProjectCatalogItem]` | `current_agent, db` | 代理项目目录（带准入策略） |
| `GET` | `/api/agents/my/project-access/requests` | `my_project_auth_requests` | `AgentProjectAuthRequestListResponse` | `current_agent, db` | 我的项目开通申请 |
| `POST` | `/api/agents/my/project-access/requests` | `create_project_auth_request` | `AgentProjectAuthRequestResponse` | `current_agent, db` | 提交项目开通申请 / 满足条件时自动开通 |
| `POST` | `/api/agents/my/project-access/requests/{request_id}/cancel` | `cancel_project_auth_request` | `AgentProjectAuthRequestResponse` | `current_agent, db` | 取消我的待审核项目申请 |

## 代理管理（agents.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/agents/` | `list_agents_endpoint` | `AgentListResponse` | `current_admin, db` |  |
| `POST` | `/api/agents/` | `create_agent_endpoint` | `AgentResponse` | `current_admin, db` |  |
| `POST` | `/api/agents/auth/login` | `agent_login_endpoint` | `AgentLoginResponse` | `db` |  |
| `GET` | `/api/agents/me` | `get_my_profile` | `` | `current_agent, db` | 代理个人主页 |
| `GET` | `/api/agents/my-projects` | `get_my_authorized_projects` | `list[dict]` | `current_agent, db` |  |
| `GET` | `/api/agents/scope/list` | `list_agents_in_scope_endpoint` | `AgentFlatListResponse` | `current_agent, db` |  |
| `GET` | `/api/agents/tree` | `get_full_agent_tree` | `list[AgentSubtreeResponse]` | `current_admin, db` |  |
| `GET` | `/api/agents/{agent_id}` | `get_agent_endpoint` | `AgentResponse` | `current_admin, db` |  |
| `PATCH` | `/api/agents/{agent_id}` | `update_agent_endpoint` | `AgentResponse` | `current_admin, db` |  |
| `GET` | `/api/agents/{agent_id}/subtree` | `get_agent_subtree_endpoint` | `AgentSubtreeResponse` | `current_admin, db` |  |

## 设备数据（device.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/device/data` | `device_data` | `DeviceDataResponse` | `current_user, game_project_code, main_db, redis` |  |
| `POST` | `/api/device/heartbeat` | `heartbeat` | `HeartbeatResponse` | `current_user, game_project_code, main_db, redis` |  |
| `POST` | `/api/device/imsi` | `upload_device_imsi` | `ImsiUploadResponse` | `current_user, main_db` | 上传设备 IMSI |
| `GET` | `/api/device/list` | `list_devices` | `DeviceListResponse` | `current_user, game_project_code, main_db, redis` |  |

## 脚本参数（params.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/params/get` | `get_params` | `ParamsGetResponse` | `current_user, game_project_code` | 拉取脚本参数 |
| `POST` | `/api/params/set` | `set_params` | `ParamsSetResponse` | `current_user, game_project_code` | 保存脚本参数 |

## 热更新（update.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/update/check` | `check_update_endpoint` | `UpdateCheckResponse` | `current_user, game_project_code, redis` | 检查版本更新 |
| `GET` | `/api/update/download` | `download_update_endpoint` | `UpdateDownloadResponse` | `current_user, game_project_code, redis` | 获取下载链接 |

## 管理后台（admin.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `DELETE` | `/admin/api/agents/{agent_id}` | `delete_agent` | `` | `_, db` |  |
| `POST` | `/admin/api/auth/login` | `admin_login_endpoint` | `AdminLoginResponse` | `db` |  |
| `GET` | `/admin/api/dashboard` | `admin_dashboard` | `` | `current_admin, db` |  |
| `GET` | `/admin/api/debug/device-bindings/{user_id}` | `debug_device_bindings` | `` | `_, db` |  |
| `GET` | `/admin/api/login-logs/` | `list_login_logs` | `` | `_, db` |  |
| `DELETE` | `/admin/api/projects/{project_id}` | `delete_project` | `` | `_, db` |  |
| `DELETE` | `/admin/api/users/{user_id}` | `delete_user` | `` | `_, db` |  |
| `GET` | `/admin/api/users/{user_id}/devices` | `get_user_devices` | `` | `_, db` |  |
| `DELETE` | `/admin/api/users/{user_id}/devices/{binding_id}` | `unbind_device` | `` | `_, db` |  |

## 项目管理（projects.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/admin/api/agents/{agent_id}/project-auths/` | `list_agent_auths_endpoint` | `list[AgentProjectAuthResponse]` | `_, db` |  |
| `POST` | `/admin/api/agents/{agent_id}/project-auths/` | `grant_agent_auth_endpoint` | `AgentProjectAuthResponse` | `_, db` |  |
| `DELETE` | `/admin/api/agents/{agent_id}/project-auths/{auth_id}` | `revoke_agent_auth_endpoint` | `` | `_, db` |  |
| `PATCH` | `/admin/api/agents/{agent_id}/project-auths/{auth_id}` | `update_agent_auth_endpoint` | `AgentProjectAuthResponse` | `_, db` |  |
| `GET` | `/admin/api/projects/` | `list_projects_endpoint` | `ProjectListResponse` | `_, db` |  |
| `POST` | `/admin/api/projects/` | `create_project_endpoint` | `ProjectResponse` | `_, db` |  |
| `GET` | `/admin/api/projects/{project_id}` | `get_project_endpoint` | `ProjectResponse` | `_, db` |  |
| `PATCH` | `/admin/api/projects/{project_id}` | `update_project_endpoint` | `ProjectResponse` | `_, db` |  |

## 热更新管理（update_admin.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `POST` | `/admin/api/updates/{project_id}/{client_type}` | `upload_version_endpoint` | `` | `_, db, redis` | 发布热更新包 |
| `GET` | `/admin/api/updates/{project_id}/{client_type}/history` | `get_version_history` | `` | `_, db` | 版本历史 |
| `GET` | `/admin/api/updates/{project_id}/{client_type}/latest` | `get_latest_version` | `` | `_, db` | 获取当前活跃版本 |

## 设备监控（device_admin.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/admin/api/devices/` | `list_all_devices` | `` | `caller, main_db, redis` | 全平台设备总览（T021） |
| `GET` | `/admin/api/devices/{game_project_code}` | `list_devices_admin` | `` | `caller, main_db, redis` | 管理后台设备监控（T026） |

## 统计数据（stats.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/api/stats/agents/my/summary` | `agent_project_summary` | `AgentProjectSummaryResponse` | `current_agent, db` |  |
| `GET` | `/api/stats/platform` | `platform_summary` | `PlatformSummaryResponse` | `db, redis, _admin` |  |
| `GET` | `/api/stats/users/{user_id}/projects` | `user_project_stats` | `UserProjectStatsResponse` | `db, _admin` |  |

## 点数管理（balance_admin.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/admin/api/agents-full` | `agents_full_list` | `` | `_, db` | 代理列表（含余额和授权项目） |
| `GET` | `/admin/api/agents/{agent_id}/balance` | `get_balance` | `` | `_, db` | 查询代理余额 |
| `POST` | `/admin/api/agents/{agent_id}/credit` | `credit` | `` | `current_admin, db` | 给代理授信 |
| `POST` | `/admin/api/agents/{agent_id}/freeze` | `freeze` | `` | `current_admin, db` | 冻结授信点数 |
| `POST` | `/admin/api/agents/{agent_id}/recharge` | `recharge` | `` | `current_admin, db` | 给代理充值点数 |
| `GET` | `/admin/api/agents/{agent_id}/transactions` | `transactions` | `` | `_, db` | 查询某代理流水记录 |
| `POST` | `/admin/api/agents/{agent_id}/unfreeze` | `unfreeze` | `` | `current_admin, db` | 解冻授信点数 |
| `GET` | `/admin/api/balance-transactions` | `global_balance_transactions` | `` | `_, db` | 管理员全局点数流水 |
| `GET` | `/admin/api/prices/{project_id}` | `get_prices` | `` | `_, db` | 获取项目各级别定价 |
| `DELETE` | `/admin/api/prices/{project_id}/{user_level}` | `delete_price` | `` | `_, db` | 删除单价 |
| `PUT` | `/admin/api/prices/{project_id}/{user_level}` | `set_price` | `` | `_, db` | 设置/更新单价 |

## 项目准入管理（project_access_admin.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/admin/api/project-access/policies` | `project_access_policies` | `` | `_, db` | 项目准入策略列表 |
| `PATCH` | `/admin/api/project-access/policies/{project_id}` | `update_policy` | `ProjectAccessPolicyResponse` | `_, db` | 更新项目准入策略 |
| `GET` | `/admin/api/project-access/requests` | `project_auth_requests` | `AgentProjectAuthRequestListResponse` | `_, db` | 代理项目开通申请列表 |
| `POST` | `/admin/api/project-access/requests/{request_id}/approve` | `approve_request` | `AgentProjectAuthRequestResponse` | `current_admin, db` | 批准代理项目开通申请 |
| `POST` | `/admin/api/project-access/requests/{request_id}/reject` | `reject_request` | `AgentProjectAuthRequestResponse` | `current_admin, db` | 拒绝代理项目开通申请 |

## 代理业务管理（agent_profile_admin.py）

| 方法 | 路径 | 函数 | 响应模型 | 依赖参数 | 说明 |
|---|---|---|---|---|---|
| `GET` | `/admin/api/agent-level-policies` | `agent_level_policies` | `list[AgentLevelPolicyAdminResponse]` | `_, db` | 代理等级策略列表 |
| `PATCH` | `/admin/api/agent-level-policies/{level}` | `update_level_policy` | `AgentLevelPolicyAdminResponse` | `_, db` | 更新代理等级策略 |
| `GET` | `/admin/api/agents/{agent_id}/business-profile` | `agent_business_profile` | `AgentBusinessProfileResponse` | `_, db` | 查询代理业务画像 |
| `PATCH` | `/admin/api/agents/{agent_id}/business-profile` | `update_business_profile` | `AgentBusinessProfileResponse` | `_, db` | 更新代理业务画像 |
| `POST` | `/admin/api/agents/{agent_id}/password` | `reset_password` | `AgentPasswordResetResponse` | `_, db` | 管理员重置代理密码 |

## 路由注册顺序风险

已确认：`app/main.py` 中有显式注释要求 `balance_agent.router` 必须先于 `agents.router` 注册，否则 `/api/agents/catalog` 这类静态路径可能被 `/api/agents/{agent_id}` 动态路由吞掉。

建议方案：把“静态路径优先于动态路径”写入 API 路由规范，后续新增 `/api/agents/*` 时必须检查注册顺序。