---
文件位置: 01-网络验证系统/Token版本号与全设备即时踢出方案.md
名称: Verify Token 版本号与全设备即时踢出方案
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-29
版本: V1.1.0
状态: 已落地
---

> **实施状态：✅ 已落地（2026-05-09 审查确认）。**
>
> 本文档描述的 token_version 方案已在源码完整实现：
> - `models.py:260` — `user.token_version` 字段 + `CheckConstraint`
> - `security.py:61` — `create_access_token` 签发时嵌入 `token_version`
> - `auth_service.py:255,370,485` — 登录/刷新签发 + revoke_all 递增
> - `dependencies.py:115-117` — `get_current_user` 校验 `payload.token_version == user.token_version`
> - `redis_client.py:275,293` — RT v2 存储含 `token_version`
> - `test_auth.py` — `test_revoke_all_invalidates_all_access_tokens` 验证即时踢出
>
> 下文保留作为方案设计参考。
关联文档:
  - "[[项目总大纲]]"
  - "[[01-网络验证系统/架构设计]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/风险与待决策清单]]"
  - "[[01-网络验证系统/测试基线]]"
  - "[[01-网络验证系统/三端联调测试清单]]"
  - "[[02-PC中控框架/Verify接口调用清单]]"
  - "[[03-安卓脚本框架/Verify接口调用清单]]"
  - "[[编码规范]]"
  - "[[编码规范_补充_Markdown文档规范]]"
变更记录:
  - V1.0.0: 基于 HiveGreatSage-Verify 当前 auth.py、auth_service.py、redis_client.py、security.py、dependencies.py 反向整理当前 Token 吊销能力、revoke-all 局限、token_version 改造方案、测试清单与迁移路线
---

# Verify Token 版本号与全设备即时踢出方案

← 返回 [[项目总大纲]] | 父节点: [[01-网络验证系统/架构设计]]

## 一、文档目的

本文用于沉淀 `HiveGreatSage-Verify` 当前 **Access Token / Refresh Token / 黑名单 / 踢出所有设备** 的真实状态，并提出后续 `token_version` 改造方案。

本文重点回答：

```text
1. 当前 Verify Token 体系已经实现了什么？
2. 当前 /api/auth/revoke-all 到底能踢出什么？
3. 为什么当前 revoke-all 不能保证所有 Access Token 即时失效？
4. Redis 黑名单和 Refresh Token 删除分别解决什么问题？
5. token_version 应该加在哪里？
6. token_version 如何让所有已签发 Access Token 即时失效？
7. PCControl / AndroidScript 遇到 401 后应如何处理？
8. 迁移需要改哪些模型、服务、依赖、测试？
9. 哪些测试必须补齐后才能对外宣称“全设备即时踢出”？
```

---

## 二、当前结论

### 2.1 已确认

当前 Verify 已经实现基础 Token 体系。

已确认源码层面存在：

```text
1. POST /api/auth/login
   用户登录，签发 Access Token 和 Refresh Token。

2. POST /api/auth/refresh
   使用 Refresh Token 换取新的 Access Token。

3. POST /api/auth/logout
   登出，当前 Access Token 加入黑名单，并删除对应 Refresh Token。

4. POST /api/auth/revoke-all
   踢出所有设备，当前 Access Token 加入黑名单，并删除该用户所有 Refresh Token。

5. GET /api/auth/me
   获取当前用户信息。

6. Redis Access Token 黑名单:
   token:blacklist:{jti}

7. Redis Refresh Token 主 Key:
   refresh:{user_id}:{jti}

8. Redis Refresh Token 反查索引:
   rt_lookup:{sha256(rt_value)[:32]}

9. get_current_user 会校验:
   JWT 签名与有效期
   Redis 黑名单
   用户存在且 active
```

### 2.2 当前 revoke-all 的真实能力

当前 `/api/auth/revoke-all` 能做到：

```text
1. 让当前请求携带的 Access Token 立即失效。
2. 删除该用户所有 Refresh Token 主 Key。
3. 让所有设备无法再通过 Refresh Token 刷新新的 Access Token。
```

当前 `/api/auth/revoke-all` 不能做到：

```text
1. 让其他设备已经签发的 Access Token 立即失效。
2. 扫描并吊销该用户所有历史 Access Token。
3. 让所有正在运行的 AndroidScript 心跳下一秒必定失败。
4. 让所有 PCControl 立即弹出重新登录。
```

原因：

```text
Access Token 是 Stateless JWT。
服务端默认不知道某个用户当前签发过多少个仍未过期的 Access Token。
当前 Redis 黑名单只记录被主动吊销的 jti。
revoke-all 当前只知道“当前这个 Access Token”的 jti。
```

### 2.3 当前风险判断

当前状态应标记为：

```text
源码已实现:
  logout / revoke-all / Refresh Token 删除 / 当前 AT 黑名单。

待修订:
  真正意义上的所有 Access Token 即时失效。

风险等级:
  P1 高风险。
```

### 2.4 推荐结论

后续应引入：

```text
user.token_version
```

并在 Access Token payload 中加入：

```text
token_version
```

鉴权时比较：

```text
payload.token_version == user.token_version
```

当用户执行：

```text
POST /api/auth/revoke-all
```

服务端执行：

```text
1. user.token_version += 1。
2. 删除该用户所有 Refresh Token。
3. 当前 Access Token 加黑名单。
```

这样所有旧 Access Token 即使没有进入黑名单，也会因为 `token_version` 不匹配而立即失效。

---

## 三、术语定义

## 3.1 Access Token

Access Token 是用户访问 Verify 用户态接口的短期 JWT。

当前 payload 包含：

```text
sub:
  用户 ID。

level:
  用户等级。

project:
  当前登录项目 code_name。

jti:
  Token 唯一 ID，用于黑名单。

iat:
  签发时间。

exp:
  过期时间。

type:
  access。
```

当前有效期：

```text
由 settings.ACCESS_TOKEN_EXPIRE_MINUTES 控制。
当前源码注释口径为 15 分钟。
```

---

## 3.2 Refresh Token

Refresh Token 是不透明随机字符串，本身不携带业务信息。

当前存储在 Redis 中：

```text
refresh:{user_id}:{jti}
```

并建立反查索引：

```text
rt_lookup:{sha256(rt_value)[:32]}
```

用途：

```text
客户端只提交 refresh_token。
服务端通过 rt_lookup 快速定位 refresh:{user_id}:{jti}。
避免 refresh 接口 SCAN 全表。
```

---

## 3.3 jti

`jti` 是 Access Token 的唯一 ID。

用途：

```text
1. 放入 Access Token payload。
2. 与 Refresh Token 主 Key 绑定。
3. 登出或吊销时写入 Redis 黑名单。
```

当前黑名单 Key：

```text
token:blacklist:{jti}
```

---

## 3.4 Redis 黑名单

Redis 黑名单用于记录已经被吊销的 Access Token jti。

当前 Key：

```text
token:blacklist:{jti}
```

Value：

```text
revoked
```

TTL：

```text
Access Token 剩余有效期。
```

用途：

```text
1. logout 后当前 Access Token 不再可用。
2. revoke-all 后当前 Access Token 不再可用。
```

限制：

```text
黑名单只能拦截已经知道 jti 的 Token。
```

---

## 3.5 token_version

`token_version` 是建议新增的用户级 Token 版本号。

字段位置建议：

```text
user.token_version
```

默认值：

```text
0
```

含义：

```text
当前用户所有有效 Access Token 必须携带与 user.token_version 相同的版本号。
```

当需要踢出所有设备时：

```text
user.token_version += 1
```

结果：

```text
旧 Access Token payload.token_version 仍是旧值。
鉴权时发现不一致。
立即返回 401。
```

---

## 四、当前 Token 流程

## 4.1 登录流程

接口：

```text
POST /api/auth/login
```

当前流程：

```text
1. 用户名存在。
2. 密码匹配。
3. 用户状态 active。
4. 用户账号未过期。
5. project_uuid 对应项目存在且 active。
6. 用户对该项目有 active Authorization。
7. Authorization 未过期。
8. Android 设备绑定检查；PC 跳过绑定。
9. 签发 Access Token。
10. 生成 Refresh Token。
11. Refresh Token 写入 Redis。
12. 写成功登录日志。
13. 返回 Access Token / Refresh Token。
```

当前 Access Token 创建：

```text
create_access_token(
  user_id=user.id,
  user_level=user.user_level,
  game_project_code=game_project.code_name,
)
```

当前缺少：

```text
token_version
```

---

## 4.2 Refresh 流程

接口：

```text
POST /api/auth/refresh
```

当前流程：

```text
1. 通过 rt_lookup 反查 Refresh Token 主 Key。
2. 读取 user_id / jti / device_fingerprint / game_project_code。
3. 查询 User。
4. 用户存在且 active。
5. 签发新的 Access Token。
6. 返回新的 Access Token。
```

当前特点：

```text
1. Refresh Token 本身有效期内可多次使用。
2. Phase 1 不做滚动刷新。
3. refresh 时不会删除旧 Access Token。
4. refresh 时不会校验 token_version，因为当前还没有 token_version。
```

---

## 4.3 Logout 流程

接口：

```text
POST /api/auth/logout
```

当前流程：

```text
1. 解码当前 Access Token。
2. 读取 jti / exp / user_id。
3. 计算 Access Token 剩余有效时间。
4. 将当前 jti 写入 token:blacklist:{jti}。
5. 删除 refresh:{user_id}:{jti}。
6. 删除对应 rt_lookup:{hash}。
7. 返回登出成功。
```

当前能力：

```text
当前 Access Token 立即失效。
当前 Refresh Token 立即失效。
```

当前限制：

```text
不影响其他设备的 Access Token。
不影响其他设备的 Refresh Token。
```

这是合理的 logout 语义。

---

## 4.4 Revoke-All 流程

接口：

```text
POST /api/auth/revoke-all
```

当前流程：

```text
1. 解码当前 Access Token。
2. 读取当前 jti / exp / user_id。
3. 将当前 jti 写入 token:blacklist:{jti}。
4. 删除该用户所有 refresh:{user_id}:*。
5. 返回清除会话数量。
```

当前能力：

```text
1. 当前 Access Token 立即失效。
2. 该用户所有 Refresh Token 失效。
3. 所有设备无法再 refresh。
```

当前限制：

```text
其他设备已签发且未过期的 Access Token 仍然有效。
直到它们自然过期，或者访问接口时被其他机制拒绝。
```

---

## 4.5 当前鉴权流程

函数：

```text
get_current_user
```

当前流程：

```text
1. 从 Authorization Header 取 Bearer Token。
2. decode_access_token 校验签名和 exp。
3. 检查 payload.type == access。
4. 读取 jti。
5. 检查 Redis token:blacklist:{jti}。
6. 查询 User。
7. User 必须存在且 status=active。
8. 返回 User。
```

当前缺少：

```text
payload.token_version 与 user.token_version 对比。
```

---

## 五、当前风险

## R001：revoke-all 名称容易被误解

当前接口名：

```text
/api/auth/revoke-all
```

中文文案：

```text
踢出所有设备
```

实际能力：

```text
删除所有 Refresh Token。
当前 Access Token 立即失效。
其他 Access Token 最多继续有效到过期。
```

风险：

```text
用户或前端误以为所有设备会立即断开。
```

建议短期文案：

```text
已使所有设备无法续期登录；部分设备会在 Access Token 过期前保持短时在线。
```

不建议短期文案：

```text
所有设备已立即下线。
```

---

## R002：其他设备 Access Token 仍可访问接口

场景：

```text
1. Android A 登录，获得 AT_A。
2. PCControl 登录，获得 AT_PC。
3. PCControl 调用 revoke-all。
4. AT_PC 被加入黑名单。
5. 所有 RT 被删除。
6. Android A 的 AT_A 未进入黑名单。
7. Android A 在 AT_A 过期前继续上报心跳或拉参数。
```

影响：

```text
安全敏感场景下，账号被盗后无法做到即时强制下线所有设备。
```

---

## R003：Refresh Token 反查索引自然过期而非同步删除

当前 revoke-all 删除：

```text
refresh:{user_id}:*
```

但不额外扫描删除：

```text
rt_lookup:*
```

源码说明：

```text
反查索引 TTL 与主 Key 一致，主 Key 删除后反查索引自然过期。
```

风险：

```text
短期内 rt_lookup 可能仍指向已删除的主 Key。
```

当前影响：

```text
refresh 时先查 rt_lookup，再查 refresh 主 Key。
如果主 Key 不存在，会返回无效。
因此安全上可接受。
```

建议：

```text
保留当前方案。
不为了清理 rt_lookup:* 做全表扫描。
```

---

## R004：Admin / Agent Token 当前不在黑名单体系中

当前 User Access Token 走：

```text
get_current_user → Redis 黑名单。
```

当前 Admin / Agent Token 走：

```text
decode_admin_token / decode_agent_token
查询 Admin / Agent active
```

当前未确认：

```text
Admin / Agent 是否也需要 token_version。
```

建议：

```text
第一阶段先解决 User Token。
Admin / Agent token_version 后续单独设计。
```

原因：

```text
1. PCControl / AndroidScript 使用 User Token。
2. 三端联调主线最先需要用户设备踢出。
3. Admin / Agent 有不同有效期与安全策略。
```

---

## 六、目标设计

## 6.1 目标能力

引入 `token_version` 后，`/api/auth/revoke-all` 应实现：

```text
1. 当前用户所有已签发 Access Token 立即失效。
2. 当前用户所有 Refresh Token 立即失效。
3. 当前请求的 Access Token 立即失效。
4. PCControl 下一次请求立即收到 401。
5. AndroidScript 下一次心跳或拉参数立即收到 401。
6. 客户端收到 401 后必须清理本地 Token 并重新登录。
```

---

## 6.2 数据库字段

建议给 `user` 表新增：

```text
token_version INTEGER NOT NULL DEFAULT 0
```

字段语义：

```text
用户当前 Token 版本号。
```

约束：

```text
token_version >= 0
```

### 是否需要 updated_at

当前 User 已有：

```text
updated_at
```

revoke-all 时更新 token_version 后，`updated_at` 可同步变化。

---

## 6.3 Access Token payload

新增字段：

```text
token_version
```

目标 payload：

```json
{
  "sub": "1001",
  "level": "normal",
  "project": "game_001",
  "jti": "uuid-v4",
  "token_version": 3,
  "iat": 1777420800,
  "exp": 1777421700,
  "type": "access"
}
```

---

## 6.4 登录签发规则

登录时：

```text
读取 user.token_version。
签发 Access Token 时写入 token_version。
```

伪代码：

```python
access_token, jti = create_access_token(
    user_id=user.id,
    user_level=auth.user_level,
    game_project_code=game_project.code_name,
    token_version=user.token_version,
)
```

注意：

```text
长期 user_level 也应来自 Authorization.user_level。
当前本文只聚焦 token_version。
```

---

## 6.5 Refresh 签发规则

Refresh 时：

```text
1. 通过 Refresh Token 找到 user。
2. 确认 user.status = active。
3. 读取 user.token_version。
4. 签发新 Access Token 时写入当前 token_version。
```

如果 revoke-all 已经删除所有 Refresh Token：

```text
旧 Refresh Token 无法进入签发流程。
```

---

## 6.6 鉴权校验规则

在 `get_current_user` 中新增：

```text
payload.token_version 必须等于 user.token_version。
```

校验顺序建议：

```text
1. decode_access_token。
2. 检查 Redis 黑名单 jti。
3. 查询 User。
4. User 必须 active。
5. 检查 payload.token_version == user.token_version。
```

如果不匹配：

```text
401 Unauthorized
detail = "Token 已失效，请重新登录"
```

---

## 6.7 revoke-all 新规则

新版 `/api/auth/revoke-all`：

```text
1. 解码当前 Access Token。
2. 查询 User。
3. user.token_version += 1。
4. 当前 Access Token jti 加黑名单。
5. 删除该用户所有 Refresh Token 主 Key。
6. 提交数据库事务。
7. 返回清理结果。
```

关键点：

```text
token_version 增加后，所有旧 AT 立即失效。
删除 RT 后，所有设备无法刷新。
当前 AT 加黑名单作为额外保险。
```

---

## 6.8 logout 是否增加 token_version

不建议。

logout 语义：

```text
只退出当前设备。
```

因此 logout 应保持：

```text
1. 当前 AT 黑名单。
2. 当前 RT 删除。
3. 不修改 user.token_version。
```

否则 logout 会误伤其他设备。

---

## 七、迁移方案

## 7.1 Phase A：数据库加字段

新增 Alembic 迁移：

```text
user.token_version INTEGER NOT NULL DEFAULT 0
```

SQL 草案：

```sql
ALTER TABLE "user"
ADD COLUMN token_version INTEGER NOT NULL DEFAULT 0;

ALTER TABLE "user"
ADD CONSTRAINT chk_user_token_version_non_negative
CHECK (token_version >= 0);
```

---

## 7.2 Phase B：签发兼容

修改：

```text
app/core/security.py
```

`create_access_token` 增加可选参数：

```python
token_version: int = 0
```

payload 增加：

```python
"token_version": token_version
```

短期兼容：

```text
如果旧调用未传 token_version，则默认 0。
```

---

## 7.3 Phase C：登录和 Refresh 改造

修改：

```text
app/services/auth_service.py
```

登录：

```text
create_access_token(..., token_version=user.token_version)
```

Refresh：

```text
create_access_token(..., token_version=user.token_version)
```

---

## 7.4 Phase D：鉴权校验兼容

修改：

```text
app/core/dependencies.py
```

兼容策略：

```text
payload_version = payload.get("token_version", 0)
```

然后比较：

```text
payload_version == user.token_version
```

这样旧 Token 在迁移刚上线时：

```text
如果 user.token_version = 0，旧 Token 仍可用。
```

但执行过 revoke-all 后：

```text
user.token_version = 1
旧 Token payload 默认 0
立即失效。
```

---

## 7.5 Phase E：revoke-all 改造

修改：

```text
revoke_all_devices
```

当前函数没有传入 db。

需要改为：

```python
async def revoke_all_devices(
    access_token_str: str,
    db: AsyncSession,
    redis: aioredis.Redis,
) -> MessageResponse:
```

路由也要注入：

```python
db: AsyncSession = Depends(get_main_db)
```

执行：

```text
1. 查询 User。
2. token_version += 1。
3. 删除 Refresh Token。
4. 当前 AT 黑名单。
5. commit。
```

---

## 7.6 Phase F：测试与文档同步

必须同步：

```text
1. tests/test_auth.py。
2. tests/test_agents_recursive.py 中 revoke-all 相关测试。
3. PCControl Verify 接口调用清单。
4. AndroidScript Verify 接口调用清单。
5. API 路由清单。
6. 三端联调测试清单。
```

---

## 八、服务端修改清单

## 8.1 `app/models/main/models.py`

User 模型新增：

```python
token_version = Column(Integer, nullable=False, default=0, server_default="0")
```

建议增加 CheckConstraint：

```python
CheckConstraint("token_version >= 0", name="chk_user_token_version_non_negative")
```

---

## 8.2 `app/core/security.py`

修改函数签名：

```python
def create_access_token(
    user_id: int,
    user_level: str,
    game_project_code: str,
    token_version: int = 0,
) -> tuple[str, str]:
```

payload 增加：

```python
"token_version": token_version
```

---

## 8.3 `app/services/auth_service.py`

### 登录

修改：

```python
access_token, jti = create_access_token(
    user_id=user.id,
    user_level=user.user_level,
    game_project_code=game_project.code_name,
    token_version=user.token_version,
)
```

### Refresh

修改：

```python
access_token, _ = create_access_token(
    user_id=user.id,
    user_level=user.user_level,
    game_project_code=game_project_code,
    token_version=user.token_version,
)
```

### revoke-all

新增：

```text
查询 User。
user.token_version += 1。
commit。
```

---

## 8.4 `app/core/dependencies.py`

`get_current_user` 增加：

```python
payload_version = int(payload.get("token_version", 0))
if payload_version != user.token_version:
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Token 已失效，请重新登录",
        headers={"WWW-Authenticate": "Bearer"},
    )
```

注意：

```text
必须在查到 user 之后比较。
```

---

## 8.5 `app/schemas/auth.py`

如果登录响应需要告诉客户端当前版本，可选增加：

```text
token_version
```

但不建议客户端依赖它。

原因：

```text
token_version 是服务端安全字段。
客户端只需要处理 401。
```

建议：

```text
不暴露 token_version。
```

---

## 九、客户端处理规则

## 9.1 PCControl

PCControl 遇到 401：

```text
1. 如果是普通接口，例如 /api/device/list：
   先尝试 Refresh Token。

2. 如果 Refresh Token 成功：
   保存新 Access Token。
   重试原请求一次。

3. 如果 Refresh Token 失败：
   停止 SyncWorker。
   清理本地 Access Token。
   弹出重新登录。
```

执行 revoke-all 后：

```text
1. 旧 Access Token 因 token_version 不匹配返回 401。
2. Refresh Token 已被删除，refresh 返回 401。
3. PCControl 弹出重新登录。
```

### UI 文案建议

```text
登录状态已失效，请重新登录。
```

不要写：

```text
服务器错误。
```

---

## 9.2 AndroidScript

AndroidScript 遇到 401：

```text
1. 心跳接口 401:
   调用 Verify.ensure_token。

2. ensure_token refresh 失败:
   尝试重新登录。

3. 重新登录失败:
   停止心跳。
   toast 提示重新登录。
   可退出脚本或进入登录界面。
```

执行 revoke-all 后：

```text
1. 旧 Access Token 因 token_version 不匹配返回 401。
2. Refresh Token 已被删除，refresh 失败。
3. AndroidScript 必须重新登录。
```

---

## 9.3 不建议客户端感知 token_version

客户端不需要：

```text
读取 token_version。
保存 token_version。
比较 token_version。
```

客户端只需要：

```text
遇到 401 → refresh → refresh 失败则重新登录。
```

---

## 十、测试清单

## 10.1 当前行为基线测试

在改造前先记录当前行为：

```text
□ 用户在 PCControl 登录，获得 AT_PC / RT_PC。
□ 用户在 AndroidScript 登录，获得 AT_A / RT_A。
□ PCControl 调用 revoke-all。
□ AT_PC 立即失效。
□ RT_PC 失效。
□ RT_A 失效。
□ AT_A 在未过期前仍可访问接口。
```

该测试用于证明当前局限。

---

## 10.2 token_version 改造后测试

```text
□ 用户登录，Access Token payload.token_version = user.token_version。
□ Refresh 后新 Access Token payload.token_version = user.token_version。
□ get_current_user 校验 token_version。
□ token_version 不匹配时返回 401。
□ revoke-all 后 user.token_version + 1。
□ revoke-all 后所有旧 Access Token 均返回 401。
□ revoke-all 后所有 Refresh Token 均返回 401。
□ revoke-all 后重新登录成功。
□ 重新登录后新 Access Token 可访问接口。
```

---

## 10.3 logout 不误伤其他设备

```text
前置:
  PCControl 登录。
  AndroidScript 登录。

步骤:
  PCControl 调用 logout。

期望:
  □ PCControl 当前 AT 失效。
  □ PCControl 当前 RT 失效。
  □ AndroidScript AT 仍可用。
  □ AndroidScript RT 仍可用。
  □ user.token_version 不变化。
```

---

## 10.4 revoke-all 即时失效测试

```text
前置:
  PCControl 登录。
  AndroidScript 登录。
  两端均能访问 /api/auth/me。

步骤:
  PCControl 调用 /api/auth/revoke-all。

期望:
  □ PCControl 再访问 /api/auth/me 返回 401。
  □ AndroidScript 再访问 /api/device/heartbeat 返回 401。
  □ AndroidScript refresh 返回 401。
  □ AndroidScript 重新登录后恢复。
```

---

## 10.5 多项目 Token 测试

```text
前置:
  同一用户登录 game_001。
  同一用户登录 game_002。

步骤:
  任一项目 Token 调用 revoke-all。

期望:
  □ 同一用户所有项目的旧 Access Token 都失效。
  □ 同一用户所有 Refresh Token 都失效。
```

说明：

```text
token_version 是用户级。
因此对同一用户所有项目同时生效。
```

待决策：

```text
是否需要项目级 revoke-all？
```

短期建议：

```text
不需要。
```

---

## 10.6 黑名单仍然有效测试

即使引入 token_version，仍保留黑名单测试：

```text
□ logout 后当前 Access Token 的 jti 写入 token:blacklist。
□ get_current_user 检测黑名单返回 401。
□ 黑名单 TTL 等于 Access Token 剩余有效期。
□ TTL 到期后黑名单自动消失。
```

原因：

```text
logout 是当前设备退出，不应该修改 token_version。
因此黑名单仍然必须存在。
```

---

## 十一、迁移风险与回退

## 11.1 旧 Token 兼容风险

如果上线后直接强制要求 payload 有 token_version：

```text
所有旧 Access Token 会立即 401。
```

这可能是可接受的，也可能影响体验。

建议兼容策略：

```text
payload.get("token_version", 0)
```

这样旧 Token 在 user.token_version=0 时仍可用。

---

## 11.2 数据库迁移风险

新增字段：

```text
token_version NOT NULL DEFAULT 0
```

风险较低。

仍需执行：

```text
1. Alembic 迁移。
2. 空库迁移测试。
3. 现有库迁移测试。
```

---

## 11.3 revoke-all 事务风险

revoke-all 新流程涉及：

```text
1. 数据库 token_version + 1。
2. Redis 删除 Refresh Token。
3. Redis 写当前 AT 黑名单。
```

如果数据库成功、Redis 失败：

```text
Access Token 已经因 token_version 失效。
Refresh Token 可能仍在 Redis。
但 Refresh 后签发的新 AT 会携带新 token_version。
```

风险：

```text
旧设备可能通过旧 RT 刷出新 AT。
```

因此 revoke-all 中 Redis 删除失败需要谨慎处理。

建议：

```text
1. 先删除 Refresh Token。
2. 再增加 token_version。
3. 当前 AT 加黑名单。
4. 任一步失败返回错误。
```

但也要考虑：

```text
如果删除 RT 成功、DB commit 失败，旧 AT 仍可用到过期。
```

更稳妥方案：

```text
1. DB token_version + 1。
2. commit。
3. 删除 RT。
4. 当前 AT 黑名单。
```

这样即时失效优先由 token_version 保证。

剩余问题：

```text
RT 删除失败时，旧 RT 可 refresh 出新 token_version 的 AT。
```

因此必须：

```text
revoke-all 后 Refresh Token 数据中也校验 revoked_after 或 token_version。
```

见下方增强方案。

---

## 十二、增强方案：Refresh Token 也保存 token_version

### 12.1 当前问题

如果只在 Access Token 中保存 token_version，而 Refresh Token 数据不保存 token_version：

```text
revoke-all 后如果某个旧 Refresh Token 没有被成功删除，
它可能刷新出一个带新 token_version 的新 Access Token。
```

虽然正常情况下 revoke-all 会删除所有 RT，但为了安全应补强。

---

## 12.2 推荐增强

Refresh Token Redis 数据增加：

```text
token_version
```

登录时存入：

```json
{
  "rt_value": "...",
  "user_id": 1001,
  "jti": "...",
  "device_fingerprint": "...",
  "game_project_code": "game_001",
  "token_version": 0
}
```

Refresh 时校验：

```text
rt_data.token_version == user.token_version
```

如果不一致：

```text
401 Refresh Token 已失效
```

这样即使旧 RT 未被删除，也无法刷新。

---

## 12.3 推荐最终规则

Access Token：

```text
payload.token_version 必须等于 user.token_version。
```

Refresh Token Redis 数据：

```text
rt_data.token_version 必须等于 user.token_version。
```

revoke-all：

```text
user.token_version += 1。
```

结果：

```text
所有旧 AT 立即失效。
所有旧 RT 即使没删掉，也无法刷新。
```

删除 Redis RT 仍然要做：

```text
为了释放会话和清理状态。
```

---

## 十三、建议实施顺序

```text
Step 1:
  新增 user.token_version。

Step 2:
  create_access_token 增加 token_version 参数。

Step 3:
  登录和 refresh 签发 AT 时写入 token_version。

Step 4:
  Redis Refresh Token 数据增加 token_version。

Step 5:
  refresh_access_token 校验 rt_data.token_version == user.token_version。

Step 6:
  get_current_user 校验 payload.token_version == user.token_version。

Step 7:
  revoke-all 增加 user.token_version += 1。

Step 8:
  保留 Redis 黑名单和 delete_all_refresh_tokens。

Step 9:
  补测试。

Step 10:
  三端联调。
```

---

## 十四、当前风险清单

## R001：当前 revoke-all 不能即时吊销所有 Access Token

风险等级：

```text
P1
```

处理：

```text
引入 user.token_version。
```

---

## R002：Refresh Token 若未被删除，可能刷新出新 Token

风险等级：

```text
P1
```

处理：

```text
Refresh Token Redis 数据也保存 token_version。
refresh 时校验 token_version。
```

---

## R003：接口文案过度承诺风险

风险等级：

```text
P2
```

当前接口名“踢出所有设备”容易被理解为即时下线。

处理：

```text
在 token_version 上线前，UI 文案写成“使所有设备无法续期登录”。
```

---

## R004：Admin / Agent Token 缺少同类机制

风险等级：

```text
P2
```

处理：

```text
User Token 先改。
Admin / Agent 后续单独设计 admin.token_version / agent.token_version。
```

---

## R005：客户端 401 处理不统一

风险等级：

```text
P1
```

处理：

```text
PCControl / AndroidScript 统一 401 → refresh → refresh 失败重新登录。
```

---

## 十五、建议新增测试文件

建议新增或扩展：

```text
tests/test_auth_token_version.py
```

优先级：

```text
P0
```

测试类建议：

```python
class TestTokenVersionLogin:
    pass

class TestTokenVersionRefresh:
    pass

class TestTokenVersionRevokeAll:
    pass

class TestLogoutDoesNotBumpTokenVersion:
    pass

class TestRefreshTokenVersion:
    pass

class TestClientReauthAfterRevokeAll:
    pass
```

---

## 十六、建议新增决策日志

```markdown
## D032 — Verify 引入 user.token_version 实现用户全设备即时踢出

**日期：** 2026-04-29  
**状态：** 建议确认  
**关联文档：** [[01-网络验证系统/Token版本号与全设备即时踢出方案]]

### 背景

当前 Verify 已经实现 logout 和 revoke-all。logout 会吊销当前 Access Token 并删除当前 Refresh Token。revoke-all 会吊销当前 Access Token 并删除该用户所有 Refresh Token。

但由于 Access Token 是 Stateless JWT，其他设备已经签发的 Access Token 在过期前仍然有效。当前 revoke-all 无法保证所有设备即时下线。

### 决策

给 user 表新增：

```text
token_version
```text

Access Token payload 增加：

```text
token_version
```text

Refresh Token Redis 数据增加：

```text
token_version
```text

鉴权时校验：

```text
payload.token_version == user.token_version
```text

Refresh 时校验：

```text
rt_data.token_version == user.token_version
```text

执行 revoke-all 时：

```text
user.token_version += 1
删除所有 Refresh Token
当前 Access Token 加黑名单
```text

### 理由

```text
1. 不需要扫描所有 Access Token。
2. 可让所有旧 Access Token 即时失效。
3. 可防止旧 Refresh Token 刷出新 Access Token。
4. 保留 Redis 黑名单用于 logout 当前设备。
5. 适合多端、多设备、PCControl + AndroidScript 场景。
```text

### 后续动作

```text
□ 新增 Alembic 迁移。
□ 修改 security.py。
□ 修改 auth_service.py。
□ 修改 dependencies.py。
□ 修改 redis_client.py Refresh Token 数据结构。
□ 新增 tests/test_auth_token_version.py。
□ 三端测试 revoke-all。
```text
```

```markdown
## D033 — logout 只退出当前设备，不递增 token_version

**日期：** 2026-04-29  
**状态：** 建议确认  
**关联文档：** [[01-网络验证系统/Token版本号与全设备即时踢出方案]]

### 背景

logout 与 revoke-all 语义不同。logout 是当前设备主动退出。revoke-all 是账号安全场景下踢出所有设备。

### 决策

logout 执行：

```text
1. 当前 Access Token jti 加入黑名单。
2. 删除当前 Refresh Token。
3. 不修改 user.token_version。
```text

revoke-all 执行：

```text
1. user.token_version += 1。
2. 删除该用户所有 Refresh Token。
3. 当前 Access Token jti 加入黑名单。
```text

### 理由

```text
1. logout 不应误伤其他设备。
2. revoke-all 才是全设备安全操作。
3. 黑名单和 token_version 各司其职。
```text

### 后续动作

```text
□ 测试 logout 不影响其他设备。
□ 测试 revoke-all 影响所有设备。
```text
```

---

## 十七、推荐下一步

```text
Step 1:
  把本文复制进 Obsidian:
  01-网络验证系统/Token版本号与全设备即时踢出方案.md

Step 2:
  建立决策:
  D032 — Verify 引入 user.token_version 实现用户全设备即时踢出
  D033 — logout 只退出当前设备，不递增 token_version

Step 3:
  新增测试设计:
  tests/test_auth_token_version.py

Step 4:
  先写当前行为基线测试:
  证明当前 revoke-all 无法让其他 AT 即时失效。

Step 5:
  新增 Alembic:
  user.token_version INTEGER NOT NULL DEFAULT 0

Step 6:
  修改:
  app/core/security.py
  app/services/auth_service.py
  app/core/dependencies.py
  app/core/redis_client.py

Step 7:
  重跑:
  tests/test_auth.py
  tests/test_agents_recursive.py
  tests/test_auth_token_version.py

Step 8:
  三端联调:
  PCControl revoke-all 后 AndroidScript 心跳应立即 401。

Step 9:
  更新:
  PCControl 与 AndroidScript 的 401 处理文档。
```

---

## 十八、结论

当前 Verify Token 系统已经具备基础安全能力：

```text
1. Access Token 短期有效。
2. Access Token 有 jti。
3. 当前 Access Token 可被加入 Redis 黑名单。
4. Refresh Token 存 Redis。
5. Refresh Token 有 O(1) 反查索引。
6. logout 可退出当前设备。
7. revoke-all 可删除所有 Refresh Token。
```

但当前仍不能宣称：

```text
所有设备会被立即踢下线。
```

当前最准确结论：

```text
Verify 当前 revoke-all 已实现“所有设备无法续期登录”；
但未实现“所有已签发 Access Token 即时失效”；
要实现真正的全设备即时踢出，必须引入 user.token_version，并让 Access Token 与 Refresh Token 都携带并校验该版本。
```

该方案完成后，三端行为应达到：

```text
PCControl 点击踢出所有设备
→ Verify user.token_version + 1
→ AndroidScript 下一次心跳立刻 401
→ Refresh Token 刷新失败
→ AndroidScript 重新登录
→ PCControl / AndroidScript 都不再误认为旧 Token 可用
```