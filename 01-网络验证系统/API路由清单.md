---
文件位置: 01-网络验证系统/API路由清单.md
名称: API路由清单
作者: 蜂巢·大圣 (HiveGreatSage)
时间: 2026-05-01
版本: V1.0.1
状态: 草稿
关联文档:
  - "[[01-网络验证系统/OpenAPI快照与接口契约治理规范]]"
  - "[[01-网络验证系统/接入契约]]"
  - "[[01-网络验证系统/热更新端到端测试清单]]"
  - "[[02-PC中控框架/Verify接口调用清单]]"
  - "[[03-安卓脚本框架/Verify接口调用清单]]"
变更记录:
  - V1.0.1: 同步 Verify 热更新链路小修复；明确客户端 check/download 已改为读取主库 VersionRecord，仍待运行测试验证
  - V1.0.0: Obsidian 去漂移重构生成
---

# API路由清单


## 当前定位

本文件是手写 API 导航清单，不再作为唯一接口真相源。接口字段、请求体、响应体必须以 FastAPI OpenAPI 快照为准。

## 关键接口组

### 终端 User 接口

- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `GET /api/auth/me`
- `POST /api/auth/revoke-all`
- `POST /api/device/heartbeat`
- `GET /api/device/list`
- `GET /api/device/data`
- `POST /api/device/imsi`
- `GET /api/params/get`
- `POST /api/params/set`
- `GET /api/update/check`
- `GET /api/update/download`
- `GET /api/client/network-config`

### 管理员接口

- `/admin/api/auth/*`
- `/admin/api/users/*`
- `/admin/api/agents/*`
- `/admin/api/projects/*`
- `/admin/api/devices/*`
- `/admin/api/updates/{project_id}/{client_type}`
- `/admin/api/updates/{project_id}/{client_type}/latest`
- `/admin/api/updates/{project_id}/{client_type}/history`
- `/admin/api/accounting/*`
- `/admin/api/system-settings/*`
- `/admin/api/project-access/*`

### 代理接口

- `/api/agents/auth/login`
- `/api/agents/me`
- `/api/agents/my/balance`
- `/api/agents/my/transactions`
- `/api/agents/my/project-access/*`

## 已确认漂移点

1. 旧 API实现清单中的路由数量口径与 API路由清单不一致，已合并到本文件。
2. 旧 balance 双前缀描述不再作为当前真相源，应以 balance_admin / balance_agent / accounting 当前源码为准。
3. 热更新后台上传路径应使用 `project_id`，不是 `GAME_PROJECT_CODE`。
4. 已确认源码层修订：客户端热更新 `check` / `download` 已改为通过主库 `VersionRecord` 读取活跃版本；运行结果仍需执行 `tests/test_update.py` 与端到端联调验证。

## 维护规则

每次改 routers / schemas / dependencies 后，必须重新导出 OpenAPI 快照，并同步更新本清单。
