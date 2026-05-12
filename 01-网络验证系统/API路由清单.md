---
文件位置: 01-网络验证系统/API路由清单.md
名称: API路由清单
作者: 蜂巢·大圣 (HiveGreatSage)
时间: 2026-05-12
版本: V1.2.1
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
- 本轮未重新导出 OpenAPI 快照。
- 本轮未重新运行接口测试。
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
POST /api/device/imsi

GET  /api/params/get
POST /api/params/set

GET  /api/update/check
GET  /api/update/download

GET  /api/client/network-config
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
