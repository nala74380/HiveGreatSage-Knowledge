---
文件位置: 01-网络验证系统/API鉴权方案.md
名称: API鉴权方案
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-05-06
版本: V1.1.0
状态: 草稿
关联文档:
  - "[[00-项目总控/项目总大纲]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/接入契约]]"
  - "[[01-网络验证系统/旧字段旧接口清理清单]]"
变更记录:
  - V1.1.0 (2026-05-06): 对齐当前源码；移除旧 level/project 与 Refresh Token JWT 口径；明确 User/Admin/Agent Token 类型隔离
  - V1.0.0: Obsidian 去漂移重构生成
---

# API鉴权方案

← 返回 [[项目总大纲]] | 父节点: [[01-网络验证系统/架构设计]]

## 一、当前状态

已确认：当前源码已经区分三类 Token：

```text
User Access Token: type = access
Admin Token:       type = admin
Agent Token:       type = agent
```

已确认：Refresh Token 当前不是 JWT，而是不透明随机字符串，由服务端 Redis 保存映射。

待确认：Admin / Agent Token 仍未接入服务端即时吊销闭环；该项以 [[Token版本号与全设备即时踢出方案]] 为后续方案来源。

## 二、User Access Token

| 字段 | 当前口径 |
|---|---|
| 格式 | JWT / HS256 |
| 用途 | 终端 User 接口鉴权 |
| 类型字段 | `type = access` |
| 用户字段 | `sub` |
| 授权等级字段 | `authorization_level`，来源于 `Authorization.user_level` |
| 项目字段 | `project_code`，来源于 `GameProject.code_name` |
| 会话字段 | `jti` |
| 时间字段 | `iat`, `exp` |

示例：

```json
{
  "sub": "10086",
  "authorization_level": "vip",
  "project_code": "game_001",
  "jti": "uuid-v4",
  "iat": 1713254400,
  "exp": 1713255300,
  "type": "access"
}
```

禁止继续使用旧字段作为新契约：

```text
level
project
user_level 作为 Token 字段
```

## 三、Refresh Token

已确认：当前 Refresh Token 是不透明随机字符串，不是 JWT。

| 项目 | 当前口径 |
|---|---|
| 生成 | `secrets.token_urlsafe(48)` |
| 客户端传递 | 登录返回；刷新时请求体提交 |
| 服务端存储 | Redis |
| 反查索引 | `rt_lookup:{sha256(refresh_token)[:32]}` |
| 映射内容 | user_id、jti、game_project_code 等会话信息 |

禁止继续把 Refresh Token 写成：

```text
JWT Refresh Token
HS256 Refresh Token payload
```

## 四、Admin / Agent Token

| 类型 | 字段 | 用途 |
|---|---|---|
| Admin Token | `type = admin`, `sub = admin_id` | 管理后台 |
| Agent Token | `type = agent`, `sub = agent_id` | 代理后台 |

当前已确认：`decode_admin_token()` 与 `decode_agent_token()` 会检查 `type`，不会和 User Access Token 混用。

待确认 / 待实现：

```text
Admin / Agent Token token_version
Admin / Agent Token 服务端吊销表或 Redis 黑名单
密码重置后即时踢出
单端会话管理
```

## 五、路由鉴权分层

| 身份 | 典型路径 | 说明 |
|---|---|---|
| Public | `/api/auth/login`, `/api/auth/refresh`, `/api/agents/auth/login`, `/admin/api/auth/login`, `/api/client/network-config` | 登录与公开客户端配置 |
| User Token | `/api/device/*`, `/api/params/*`, `/api/update/*`, `/api/auth/me` | 必须带项目上下文 |
| Agent Token | `/api/agents/me`, `/api/agents/my/*` | 代理自身与代理项目目录 |
| Admin Token | `/admin/api/*` | 管理后台 |
| Admin / Agent 共用 | `/api/users/*` | Admin 全量；Agent 当前只管理直属用户 |

## 六、当前风险

```text
R-AUTH-001：Admin / Agent Token 未接入服务端即时吊销。
R-AUTH-002：HTTPBearer 默认无 Token 可能返回 403，后续应统一为 401。
R-AUTH-003：前端 Admin / Agent Token 当前存 localStorage，生产前需确认 XSS 防护策略。
```

## 七、维护规则

1. Token 字段改动必须同步更新本文件、接入契约和 OpenAPI 快照。
2. 新接口不得读取旧 Token 字段 `level` / `project`。
3. Refresh Token 不得在文档中写成 JWT。
4. Admin / Agent / User 三类 Token 不得共用解码函数。
