# API鉴权方案

> **所属模块：** 01-网络验证系统 **文档状态：** 草稿（推演定稿） **最后更新：** 2026-04-16 **关联决策：** [[决策日志#JWT鉴权方案确定]]
> 
> **变更记录：**
> 
> - v1：初始版本
> - v2：跨文档统一对齐 — API 路径前缀统一为 `/api/`（去除 `/v1`）

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
|**格式**|JWT (HS256 签名)|
|**有效期**|**15 分钟**|
|**Payload 内容**|`sub`(user_id), `level`(user_level), `project`(game_code_name), `jti`, `iat`, `exp`|
|**传输方式**|HTTP Header: `Authorization: Bearer <token>`|
|**存储位置（客户端）**|PC中控：内存；安卓脚本：加密的本地文件；Web：内存或 HttpOnly Cookie|

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

|属性|值|
|---|---|
|**格式**|随机字符串（UUID v4 + 哈希），不透明字符串|
|**有效期**|7 天（可配置，支持"记住我"延长至 30 天）|
|**存储（服务端）**|Redis，Key: `refresh:{user_id}:{jti}`，Value: 关联信息|
|**传输方式**|登录时返回；刷新时通过请求体 JSON 传递|
|**存储（客户端）**|PC中控：加密的本地配置文件；安卓脚本：加密本地文件；Web：HttpOnly Cookie（推荐）|

**Refresh Token 服务端存储结构（示例）：**

```json
{
  "user_id": 10086,
  "jti": "access-token-jti",
  "device_fingerprint": "可选，绑定设备增强安全",
  "created_at": "2026-04-16T10:00:00Z",
  "expires_at": "2026-04-23T10:00:00Z"
}
```

---

## 三、认证与刷新流程

### 3.1 首次登录

```
客户端 → POST /api/auth/login { 见接入契约 }
服务端：
  1. 校验用户名密码、授权、设备（10步验证链）
  2. 生成 Access Token (AT) 和 Refresh Token (RT)
  3. 将 RT 存入 Redis：Key = refresh:{user_id}:{jti}
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
  1. 从 Redis 中查找 RT，校验存在且未过期
  2. 验证 RT 关联的用户信息
  3. 生成新的 AT（jti 更新）
  4. 更新 Redis 中 RT 关联的 jti（可选：滚动刷新，旧 RT 失效）
  5. 返回 { access_token, expires_in }
```

**滚动刷新策略：** Phase 1 暂不实施滚动刷新，RT 在有效期内可多次使用。

### 3.4 登出

```
客户端 → POST /api/auth/logout (需携带 AT)
服务端：
  1. 解析 AT 获取 jti 和 user_id
  2. 将 AT 的 jti 加入 Redis 黑名单（TTL = AT 剩余有效期）
  3. 删除 Redis 中对应的 Refresh Token
  4. 返回成功
```

---

## 四、Redis 数据结构设计

### 4.1 Refresh Token 存储

```
Key:   refresh:{user_id}:{jti}
Value: JSON 字符串，包含 device_info, created_at, expires_at
TTL:   与 RT 有效期一致（7天或30天）
```

### 4.2 Token 黑名单（登出/撤销）

```
Key:   token:blacklist:{jti}
Value: "revoked"
TTL:   AT 剩余有效期（通过 exp - iat 计算）
```

当用户修改密码或管理员禁用账号时，需将该用户所有活跃的 RT 从 Redis 中删除，并将对应的 AT 加入黑名单。

---

## 五、安全性设计

|层面|措施|
|---|---|
|传输安全|全站 HTTPS，禁止 HTTP 明文传输 Token|
|密钥管理|JWT 签名密钥通过环境变量注入，不同环境（开发/生产）隔离|
|设备绑定（可选）|登录时可记录设备指纹，刷新 Token 时校验，防止 RT 被盗用|
|并发登录限制|同一用户最多允许 5 个设备同时在线（通过 Redis 中 RT 数量控制）|
|敏感操作二次验证|修改密码、删除设备等操作需验证旧密码或再次输入密码|
|Token 泄露应对|提供"踢出所有设备"功能，删除 Redis 中该用户所有 RT，并加入黑名单|

---

## 六、FastAPI 实现要点

### 6.1 依赖项：get_current_user

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    redis: Redis = Depends(get_redis),
    db: AsyncSession = Depends(get_db)
) -> User:
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

    # 检查黑名单
    jti = payload.get("jti")
    if await redis.exists(f"token:blacklist:{jti}"):
        raise HTTPException(status_code=401, detail="Token revoked")

    user_id = payload.get("sub")
    user = await db.get(User, user_id)
    if not user or user.status != 'active':
        raise HTTPException(status_code=401, detail="User inactive")

    return user
```

### 6.2 刷新端点

```python
@router.post("/refresh")
async def refresh_token(
    refresh_token: str = Body(..., embed=True),
    redis: Redis = Depends(get_redis),
    db: AsyncSession = Depends(get_db)
):
    # 从 Redis 验证 RT
    rt_data = await redis.get(f"refresh:{user_id}:{jti}")
    if not rt_data:
        raise HTTPException(status_code=401, detail="Invalid refresh token")

    # 生成新 AT，更新 Redis 关联
    new_access_token = create_access_token(user_id, user_level)
    return {"access_token": new_access_token, "expires_in": 900}
```

### 6.3 中间件：自动刷新临近过期的 Token

```python
@app.middleware("http")
async def token_refresh_hint(request: Request, call_next):
    response = await call_next(request)
    token = request.headers.get("Authorization")
    if token:
        payload = decode_token(token)
        if payload and payload["exp"] - time.time() < 300:  # 5分钟内过期
            response.headers["X-Token-Expiring-Soon"] = "true"
    return response
```

---

## 七、API 端点定义

|方法|路径|描述|鉴权|
|---|---|---|---|
|POST|`/api/auth/login`|用户登录|无|
|POST|`/api/auth/logout`|登出|AT|
|POST|`/api/auth/refresh`|刷新 Token|RT|
|POST|`/api/auth/revoke-all`|踢出所有设备|AT|
|GET|`/api/auth/me`|获取当前用户信息|AT|

---

## 八、待办事项 / 后续推演

- [ ] 确定 JWT 签名算法：HS256 vs RS256（初期 HS256 足够，后期可切 RS256）
- [ ] 设计设备指纹生成规则（安卓端 hardware_serial 获取与备用方案）
- [ ] PC 端 Token 安全存储方案（Windows 凭据管理器 vs 加密文件）
- [ ] 懒人精灵 Lua 环境的 HTTP 请求库选型与测试
- [ ] Web 端 RT 存储方案：HttpOnly Cookie vs localStorage（安全与便利权衡）