# API鉴权方案

> **所属模块：** 01-网络验证系统  **文档状态：** ✅ 定稿（已落地）  **最后更新：** 2026-04-24  **关联决策：** [[决策日志#D001]]
>
> **变更记录：**
>
> - v1：初始版本
> - v2：跨文档统一对齐 — API 路径前缀统一为 `/api/`（去除 `/v1`）
> - v3（2026-04-24）：对齐实际代码实现 —
>   - Refresh Token 格式更正为 JWT（HS256），非随机字符串；
>   - 签名算法锁定为 HS256（见 D001）；
>   - 滚动刷新确认 Phase 1 不实施；
>   - 关闭已决策的待办项。

---

## 一、方案概述

网络验证系统的所有 API（除登录接口外）均需鉴权。采用 **JWT（JSON Web Token）** 作为核心鉴权机制，配合 **Refresh Token** 实现长期会话保持，结合 **Redis** 实现 Token 主动失效与黑名单。

### 适用客户端

|客户端类型|说明|
|---|---|
|PC 中控|PySide 6 桌面应用，长期运行，需支持"记住我"功能|
|安卓脚本|懒人精灵 Lua 环境，HTTP 请求能力有限，需简化 Token 存储|
|Web 管理后台|Vue 3 浏览器环境，可使用 HttpOnly Cookie 或 localStorage|

---

## 二、Token 设计

### 2.1 Access Token

用于日常 API 请求的身份凭证。

|属性|值|
|---|---|
|**格式**|JWT（HS256 签名，锁定，见 D001）|
|**有效期**|**15 分钟**|
|**Payload 内容**|`sub`(user_id), `level`(user_level), `project`(game_code_name), `jti`, `iat`, `exp`|
|**传输方式**|HTTP Header: `Authorization: Bearer <token>`|
|**存储位置（客户端）**|PC 中控：内存；安卓脚本：加密的本地文件；Web：内存或 HttpOnly Cookie|

**Payload 示例：**

```json
{
  "sub": "10086",
  "level": "vip",
  "project": "game_001",
  "jti": "uuid-v4",
  "iat": 1713254400,
  "exp": 1713255300
}
```

**设计说明：**

- `sub` = user_id
- `project` = game_project.code_name（对应客户端登录时传入的 project_uuid 解析后的 code_name）
- `jti`（JWT ID）用于 Refresh Token 关联和黑名单
- 不存放敏感信息（如密码、设备指纹），减少泄露风险

### 2.2 Refresh Token

用于获取新的 Access Token，实现免密续期。

> **⚠️ 注意（v3 更正）**：原文档写"随机字符串（UUID v4 + 哈希），不透明字符串"。
> 实际代码实现为 **JWT（HS256 签名）**，与 Access Token 格式一致，
> 通过 `type` 或 payload 内容区分用途。以代码为准。

|属性|值|
|---|---|
|**格式**|✅ **JWT（HS256 签名）**（v3 更正，见 `app/core/security.py`）|
|**有效期**|7 天（可配置，支持"记住我"延长至 30 天）|
|**存储（服务端）**|Redis，Key: `refresh:{user_id}:{jti}`，TTL = 7 天|
|**传输方式**|登录时返回；刷新时通过请求体 JSON 传递|
|**存储（客户端）**|PC 中控：加密的本地配置文件；安卓脚本：加密本地文件；Web：HttpOnly Cookie（推荐）|

**Refresh Token payload 示例：**

```json
{
  "sub": "10086",
  "jti": "uuid-v4",
  "iat": 1713254400,
  "exp": 1713859200
}
```

---

## 三、认证与刷新流程

### 3.1 首次登录

```
客户端 → POST /api/auth/login { 见接入契约 }
服务端：
  1. 校验用户名密码、授权、设备（10步验证链）
  2. 生成 Access Token (AT) 和 Refresh Token (RT)，均为 JWT
  3. 将 RT 的 jti 存入 Redis：Key = refresh:{user_id}:{jti}，TTL = 7天
  4. 返回 { access_token, refresh_token, expires_in }
客户端：
  - PC/安卓：安全存储 RT，AT 存内存
  - Web：RT 存入 HttpOnly Cookie，AT 存内存
```

### 3.2 请求鉴权

```
客户端每次请求携带 AT 于 Header
服务端中间件：
  1. 验证 JWT 签名和有效期
  2. 检查 Redis 黑名单（登出/撤销的 Token）
  3. 通过则放行，否则返回 401
```

### 3.3 Token 刷新

当 AT 过期，客户端使用 RT 换取新 AT。

```
客户端 → POST /api/auth/refresh { refresh_token }
服务端：
  1. 解析 RT JWT，校验签名和有效期
  2. 检查 Redis 中 refresh:{user_id}:{jti} 是否存在（防止已撤销 RT 复用）
  3. 验证 RT 关联的用户信息（用户状态是否仍为 active）
  4. 生成新的 AT（新 jti）
  5. 返回 { access_token, expires_in }
```

**滚动刷新策略（Phase 1 确认不实施）：**
RT 在有效期内可多次使用，不自动轮换。Phase 2 可按需开启：每次刷新时旧 RT 失效、签发新 RT。

### 3.4 登出

```
客户端 → POST /api/auth/logout (需携带 AT)
服务端：
  1. 解析 AT 获取 jti 和 user_id
  2. 将 AT 的 jti 加入 Redis 黑名单（TTL = AT 剩余有效期）
  3. 删除 Redis 中对应的 Refresh Token（Key: refresh:{user_id}:{jti}）
  4. 返回成功
```

---

## 四、Redis 数据结构设计

### 4.1 Refresh Token 存储

```
Key:   refresh:{user_id}:{jti}
Value: "1"（标记存在即可，实际用户信息从 JWT payload 解析）
TTL:   与 RT 有效期一致（7天或30天）
```

### 4.2 Token 黑名单（登出/撤销）

```
Key:   token:blacklist:{jti}
Value: "revoked"
TTL:   AT 剩余有效期（通过 exp - now 计算）
```

当用户修改密码或管理员禁用账号时，需将该用户所有活跃的 RT 从 Redis 中删除，并将对应的 AT 加入黑名单。

---

## 五、安全性设计

|层面|措施|
|---|---|
|传输安全|全站 HTTPS，禁止 HTTP 明文传输 Token|
|密钥管理|JWT 签名密钥通过环境变量注入，不同环境（开发/生产）隔离|
|签名算法|HS256（Phase 1 锁定）；Phase 2+ 可按需迁移至 RS256|
|设备绑定|登录时绑定设备指纹，同一用户最多绑定 N 台设备（按级别，见 D1）|
|并发登录限制|同一用户最多允许 N 个设备同时在线（通过 Redis RT 数量控制）|
|Token 泄露应对|提供"踢出所有设备"功能（Phase 2），删除 Redis 中该用户所有 RT|

---

## 六、FastAPI 实现要点（已落地，见代码）

### 6.1 依赖项：get_current_user

实现位置：`app/core/dependencies.py`

```python
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    redis: Redis = Depends(get_redis),
    db: AsyncSession = Depends(get_main_db)
) -> User:
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    jti = payload.get("jti")
    if await redis.exists(f"token:blacklist:{jti}"):
        raise HTTPException(status_code=401, detail="Token revoked")
    user = await db.get(User, int(payload["sub"]))
    if not user or user.status != "active":
        raise HTTPException(status_code=401, detail="User inactive")
    return user
```

### 6.2 刷新端点

实现位置：`app/routers/auth.py` + `app/services/auth_service.py`

### 6.3 中间件：Token 临近过期提示（Phase 2）

响应 Header 中加入 `X-Token-Expiring-Soon: true`，客户端可据此提前刷新。Phase 1 未实施。

---

## 七、API 端点定义

|方法|路径|描述|鉴权|状态|
|---|---|---|---|---|
|POST|`/api/auth/login`|用户登录|无|✅ 已实现|
|POST|`/api/auth/logout`|登出|AT|✅ 已实现|
|POST|`/api/auth/refresh`|刷新 Token|RT|✅ 已实现|
|GET|`/api/auth/me`|获取当前用户信息|AT|✅ 已实现|
|POST|`/api/auth/revoke-all`|踢出所有设备|AT|🔲 Phase 2|

---

## 八、待办事项

> **已关闭项（2026-04-24 对齐实现）：**
>
> - ~~确定 JWT 签名算法：HS256 vs RS256~~ → ✅ **已决策**：Phase 1 使用 HS256，Phase 2+ 可迁移 RS256（见 D001）
> - ~~滚动刷新策略~~ → ✅ **已决策**：Phase 1 不实施滚动刷新，RT 有效期内可多次使用

**仍待推演（Phase 2+）：**

- [ ] 设计设备指纹生成规则（安卓端 hardware_serial 获取与备用方案）— 见安卓脚本框架
- [ ] PC 端 Token 安全存储方案（Windows 凭据管理器 vs 加密文件）
- [ ] 懒人精灵 Lua 环境的 HTTP 请求库选型与测试
- [ ] Web 端 RT 存储方案：HttpOnly Cookie vs localStorage（安全与便利权衡）
- [ ] "踢出所有设备"接口（`POST /api/auth/revoke-all`）
- [ ] Token 临近过期提示中间件（`X-Token-Expiring-Soon` Header）
