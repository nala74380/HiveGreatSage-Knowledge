---
文件位置: 01-网络验证系统/API实现清单.md
名称: Verify API 实现清单
作者: 蜂巢·大圣 (HiveGreatSage)
时间: 2026-04-29
版本: V1.0.0
状态: 草稿
关联文档: ["[[01-网络验证系统/架构设计]]", "[[01-网络验证系统/API鉴权方案]]", "[[01-网络验证系统/接入契约]]"]
变更记录:
  - V1.0.0: 基于 HiveGreatSage-Verify.zip 当前实现反向同步生成
---

# Verify API 实现清单

> **文档性质：** Verify 当前实现反向同步文档。  
> **来源边界：** 基于 `HiveGreatSage-Verify.zip` 当前源码、前端、迁移、测试文件逐行阅读结果生成。  
> **安全口径：** 本文只记录“当前实现已出现的结构与行为”。未运行验证的内容标记为“待确认”；不把代码存在等同于生产可用。

---

## 一、统计

| 项目 | 数量 |
|---|---:|
| 路由注册条目（含重复挂载和 `/health`） | 75 |
| 路由文件 | 13 |
| 后端入口 | `app/main.py` |

> **注意：** `balance.router` 当前在 `app/main.py` 中被挂载到 `/admin/api` 与 `/api/agents` 两个前缀。该设计会让同一批子路径生成多组完整 URL，其中一部分可能是非预期暴露路径，已在风险章节单独标记。

---

## 二、路由注册表

| Router | Prefix | Tags |
|---|---|---|
| `auth` | `/api/auth` | 认证 |
| `users` | `/api/users` | 用户管理 |
| `agents` | `/api/agents` | 代理管理 |
| `device` | `/api/device` | 设备数据 |
| `params` | `/api/params` | 脚本参数 |
| `update` | `/api/update` | 热更新 |
| `admin` | `/admin/api` | 管理后台 |
| `projects` | `/admin/api` | 项目管理 |
| `update_admin` | `/admin/api/updates` | 热更新管理 |
| `device_admin` | `/admin/api/devices` | 设备监控 |
| `stats` | `/api/stats` | 统计数据 |
| `balance` | `/admin/api` | 点数管理 |
| `balance` | `/api/agents` | 代理余额 |

---

## 三、API 端点清单

| 方法 | 完整路径 | 实现函数 | 文件 |
|---|---|---|---|
| `GET` | `/admin/api/agents-full` | `agents_full_list` L215 | `app/routers/balance.py` |
| `DELETE` | `/admin/api/agents/{agent_id}` | `delete_agent` L236 | `app/routers/admin.py` |
| `GET` | `/admin/api/agents/{agent_id}/balance` | `get_balance` L126 | `app/routers/balance.py` |
| `POST` | `/admin/api/agents/{agent_id}/credit` | `credit` L149 | `app/routers/balance.py` |
| `POST` | `/admin/api/agents/{agent_id}/freeze` | `freeze` L160 | `app/routers/balance.py` |
| `POST` | `/admin/api/agents/{agent_id}/recharge` | `recharge` L135 | `app/routers/balance.py` |
| `GET` | `/admin/api/agents/{agent_id}/transactions` | `transactions` L181 | `app/routers/balance.py` |
| `POST` | `/admin/api/agents/{agent_id}/unfreeze` | `unfreeze` L171 | `app/routers/balance.py` |
| `POST` | `/admin/api/auth/login` | `admin_login_endpoint` L31 | `app/routers/admin.py` |
| `GET` | `/admin/api/catalog` | `price_catalog` L115 | `app/routers/balance.py` |
| `GET` | `/admin/api/dashboard` | `admin_dashboard` L43 | `app/routers/admin.py` |
| `GET` | `/admin/api/debug/device-bindings/{user_id}` | `debug_device_bindings` L123 | `app/routers/admin.py` |
| `GET` | `/admin/api/devices/` | `list_all_devices` L92 | `app/routers/device_admin.py` |
| `GET` | `/admin/api/devices/{game_project_code}` | `list_devices_admin` L240 | `app/routers/device_admin.py` |
| `GET` | `/admin/api/login-logs/` | `list_login_logs` L63 | `app/routers/admin.py` |
| `GET` | `/admin/api/my/balance` | `my_balance` L195 | `app/routers/balance.py` |
| `GET` | `/admin/api/my/transactions` | `my_transactions` L203 | `app/routers/balance.py` |
| `GET` | `/admin/api/prices/{project_id}` | `get_prices` L83 | `app/routers/balance.py` |
| `DELETE` | `/admin/api/prices/{project_id}/{user_level}` | `delete_price` L103 | `app/routers/balance.py` |
| `PUT` | `/admin/api/prices/{project_id}/{user_level}` | `set_price` L92 | `app/routers/balance.py` |
| `GET` | `/admin/api/projects/` | `list_projects_endpoint` L65 | `app/routers/projects.py` |
| `POST` | `/admin/api/projects/` | `create_project_endpoint` L55 | `app/routers/projects.py` |
| `DELETE` | `/admin/api/projects/{project_id}` | `delete_project` L252 | `app/routers/admin.py` |
| `GET` | `/admin/api/projects/{project_id}` | `get_project_endpoint` L84 | `app/routers/projects.py` |
| `PATCH` | `/admin/api/projects/{project_id}` | `update_project_endpoint` L93 | `app/routers/projects.py` |
| `GET` | `/admin/api/updates/{project_id}/{client_type}/history` | `get_version_history` L186 | `app/routers/update_admin.py` |
| `GET` | `/admin/api/updates/{project_id}/{client_type}/latest` | `get_latest_version` L165 | `app/routers/update_admin.py` |
| `DELETE` | `/admin/api/users/{user_id}` | `delete_user` L220 | `app/routers/admin.py` |
| `GET` | `/admin/api/users/{user_id}/devices` | `get_user_devices` L150 | `app/routers/admin.py` |
| `DELETE` | `/admin/api/users/{user_id}/devices/{binding_id}` | `unbind_device` L196 | `app/routers/admin.py` |
| `GET` | `/api/agents/` | `list_agents_endpoint` L241 | `app/routers/agents.py` |
| `POST` | `/api/agents/` | `create_agent_endpoint` L231 | `app/routers/agents.py` |
| `GET` | `/api/agents/agents-full` | `agents_full_list` L215 | `app/routers/balance.py` |
| `GET` | `/api/agents/agents/{agent_id}/balance` | `get_balance` L126 | `app/routers/balance.py` |
| `POST` | `/api/agents/agents/{agent_id}/credit` | `credit` L149 | `app/routers/balance.py` |
| `POST` | `/api/agents/agents/{agent_id}/freeze` | `freeze` L160 | `app/routers/balance.py` |
| `POST` | `/api/agents/agents/{agent_id}/recharge` | `recharge` L135 | `app/routers/balance.py` |
| `GET` | `/api/agents/agents/{agent_id}/transactions` | `transactions` L181 | `app/routers/balance.py` |
| `POST` | `/api/agents/agents/{agent_id}/unfreeze` | `unfreeze` L171 | `app/routers/balance.py` |
| `POST` | `/api/agents/auth/login` | `agent_login_endpoint` L69 | `app/routers/agents.py` |
| `GET` | `/api/agents/catalog` | `price_catalog` L115 | `app/routers/balance.py` |
| `GET` | `/api/agents/me` | `get_my_profile` L80 | `app/routers/agents.py` |
| `GET` | `/api/agents/my-projects` | `get_my_authorized_projects` L168 | `app/routers/agents.py` |
| `GET` | `/api/agents/my/balance` | `my_balance` L195 | `app/routers/balance.py` |
| `GET` | `/api/agents/my/transactions` | `my_transactions` L203 | `app/routers/balance.py` |
| `GET` | `/api/agents/prices/{project_id}` | `get_prices` L83 | `app/routers/balance.py` |
| `DELETE` | `/api/agents/prices/{project_id}/{user_level}` | `delete_price` L103 | `app/routers/balance.py` |
| `PUT` | `/api/agents/prices/{project_id}/{user_level}` | `set_price` L92 | `app/routers/balance.py` |
| `GET` | `/api/agents/scope/list` | `list_agents_in_scope_endpoint` L208 | `app/routers/agents.py` |
| `GET` | `/api/agents/tree` | `get_full_agent_tree` L260 | `app/routers/agents.py` |
| `GET` | `/api/agents/{agent_id}` | `get_agent_endpoint` L283 | `app/routers/agents.py` |
| `PATCH` | `/api/agents/{agent_id}` | `update_agent_endpoint` L303 | `app/routers/agents.py` |
| `GET` | `/api/agents/{agent_id}/subtree` | `get_agent_subtree_endpoint` L293 | `app/routers/agents.py` |
| `POST` | `/api/auth/login` | `login` L66 | `app/routers/auth.py` |
| `POST` | `/api/auth/logout` | `logout` L104 | `app/routers/auth.py` |
| `GET` | `/api/auth/me` | `get_me` L142 | `app/routers/auth.py` |
| `POST` | `/api/auth/refresh` | `refresh` L94 | `app/routers/auth.py` |
| `POST` | `/api/auth/revoke-all` | `revoke_all` L118 | `app/routers/auth.py` |
| `GET` | `/api/device/data` | `device_data` L136 | `app/routers/device.py` |
| `POST` | `/api/device/heartbeat` | `heartbeat` L61 | `app/routers/device.py` |
| `POST` | `/api/device/imsi` | `upload_device_imsi` L117 | `app/routers/device.py` |
| `GET` | `/api/device/list` | `list_devices` L97 | `app/routers/device.py` |
| `GET` | `/api/params/get` | `get_params` L45 | `app/routers/params.py` |
| `POST` | `/api/params/set` | `set_params` L69 | `app/routers/params.py` |
| `GET` | `/api/stats/agents/my/summary` | `agent_project_summary` L49 | `app/routers/stats.py` |
| `GET` | `/api/stats/platform` | `platform_summary` L61 | `app/routers/stats.py` |
| `GET` | `/api/stats/users/{user_id}/projects` | `user_project_stats` L36 | `app/routers/stats.py` |
| `GET` | `/api/update/check` | `check_update_endpoint` L52 | `app/routers/update.py` |
| `GET` | `/api/update/download` | `download_update_endpoint` L93 | `app/routers/update.py` |
| `GET` | `/api/users/` | `list_users_endpoint` L113 | `app/routers/users.py` |
| `POST` | `/api/users/` | `create_user_endpoint` L102 | `app/routers/users.py` |
| `GET` | `/api/users/{user_id}` | `get_user_endpoint` L136 | `app/routers/users.py` |
| `PATCH` | `/api/users/{user_id}` | `update_user_endpoint` L146 | `app/routers/users.py` |
| `DELETE` | `/api/users/{user_id}/authorizations/{auth_id}` | `revoke_auth_endpoint` L173 | `app/routers/users.py` |
| `GET` | `/health` | `health_check` | `app/main.py` |

---

## 四、重点审查项

### 4.1 `balance.router` 双前缀挂载

**已确认：** `app/main.py` 中存在两次 `balance.router` 挂载：

```python
app.include_router(balance.router, prefix="/admin/api",  tags=["点数管理"])
app.include_router(balance.router, prefix="/api/agents", tags=["代理余额"])
```

**风险：**  
如果 `balance.py` 内部路径既包含 `/agents/{agent_id}/...`，又包含 `/my/...`、`/catalog`，双前缀会生成一批非前端预期路径，例如 `/api/agents/agents/{agent_id}/balance`。这不一定立即造成错误，但会扩大 API 暴露面。

**建议方案：**

- 拆成 `balance_admin.py` 与 `balance_agent.py` 两个 router。
- Admin 路由只挂 `/admin/api`。
- Agent 自查路由只挂 `/api/agents`。
- 前端同步更新调用路径。

### 4.2 管理后台与终端客户端共存

**已确认：** Verify 同时提供：

- 终端客户端接口：`/api/auth`, `/api/device`, `/api/params`, `/api/update`
- 管理后台接口：`/admin/api/*`
- 代理接口：`/api/agents/*`

**建议方案：** 后续 API 文档应按“调用方”拆分，而不是只按路由文件拆分：

1. PC 中控 / 安卓脚本接口
2. 管理员后台接口
3. 代理后台接口
4. 系统健康与运维接口