## 1. 当前任务理解

本轮继续主线精读审查，范围是：

```text
代理管理与代理树治理
  ↓
Admin 创建/管理代理
  ↓
Agent 创建/管理下级代理
  ↓
代理树 parent_agent_id / hierarchy_depth
  ↓
代理业务等级 tier_level
  ↓
代理范围查询、越权边界、软硬删除、前端创建流程、测试覆盖
```

本轮不进入局部补丁，不漂移到旧字段专项，只把发现的问题归入主线审查台账。

---

## 2. 本轮精读进度

### 本轮新增完整精读

本轮新增完整精读 **9 个文件 / 约 2028 行**：

| 文件                                                                 |  行数 | 状态   |
| ------------------------------------------------------------------ | --: | ---- |
| `01-网络验证系统/代理管理方案.md`                                              |  92 | 完整精读 |
| `app/services/agent_service.py`                                    | 697 | 完整精读 |
| `app/core/utils.py`                                                | 145 | 完整精读 |
| `app/schemas/agent.py`                                             | 181 | 完整精读 |
| `app/routers/agent_profile_admin.py`                               | 173 | 完整精读 |
| `app/services/agent_profile_service.py`                            | 401 | 完整精读 |
| `frontend/src/api/agent.js`                                        |  69 | 完整精读 |
| `frontend/src/api/admin/agentProfile.js`                           |  32 | 完整精读 |
| `frontend/src/views/shared/agent/components/AgentCreateDialog.vue` | 238 | 完整精读 |

### 本轮联动复核

这些文件此前已计入完整精读，本轮做了专项复读，不重复计数：

```text
app/routers/agents.py
app/routers/agent_scope_management.py
app/services/user_service.py
app/models/main/models.py
app/models/main/agent_profile.py
tests/test_agents_recursive.py
frontend/src/views/shared/agent/AgentListPage.vue
frontend/src/views/shared/agent/components/AgentEditDrawer.vue
```

### 累计进度

```text
上一轮累计完整精读：122 / 293
本轮新增完整精读：9
累计完整精读：131 / 293
剩余未完整精读：162 / 293

上一轮累计精读行数：约 46,516
本轮新增精读行数：约 2,028
累计完整精读行数：约 48,544 / 103,000+
行级深读进度：约 47.1%
文件级深读进度：约 44.7%
```

---

## 3. 已确认信息

### 3.1 代理树主模型边界是清楚的

**已确认：**

当前 `Agent` 模型中：

```text
parent_agent_id = 父代理
hierarchy_depth = 代理组织层级 / 代理树深度
数据库列名仍为 level
ORM 属性名为 hierarchy_depth
```

并且文档和源码都明确区分：

```text
Agent.hierarchy_depth：组织层级 / 树深度
AgentBusinessProfile.tier_level：业务等级 Lv.1 - Lv.4
```

这一点目前主线方向正确。

---

### 3.2 代理树查询已经使用 WITH RECURSIVE

**已确认：**

`app/core/utils.py:get_agent_scope_ids()` 使用 PostgreSQL `WITH RECURSIVE` 获取当前代理及全部下级代理 ID。

`app/services/agent_service.py` 中：

```text
get_agent_subtree()
get_all_agent_ids_in_subtree()
list_agents_in_scope()
```

也都是围绕递归代理树展开。

**判断：**

代理树查询不是靠 Python 层循环查库，方向正确。

---

### 3.3 Admin / Agent 两套代理管理入口已经分开

**已确认：**

Admin 入口：

```text
POST   /api/agents/
GET    /api/agents/
GET    /api/agents/tree
GET    /api/agents/{agent_id}
GET    /api/agents/{agent_id}/subtree
PATCH  /api/agents/{agent_id}
DELETE /api/agents/{agent_id}
```

Agent 入口：

```text
POST   /api/agents/scope
GET    /api/agents/scope/list
GET    /api/agents/scope/{agent_id}
GET    /api/agents/scope/{agent_id}/subtree
PATCH  /api/agents/scope/{agent_id}
```

更细的代理范围治理入口在：

```text
/api/agents/scope/{agent_id}/business-profile
/api/agents/scope/{agent_id}/project-auths
/api/agents/scope/{agent_id}/balance
/api/agents/scope/{agent_id}/transactions
/api/agents/scope/{agent_id}/recharge
/api/agents/scope/{agent_id}/credit
/api/agents/scope/{agent_id}/freeze
/api/agents/scope/{agent_id}/unfreeze
/api/agents/scope/{agent_id}/password
```

并且 `app/main.py` 中 `agent_scope_management.router` 注册在 `agents.router` 之前，用来避免 `/scope/level-policies` 被 `/scope/{agent_id}` 抢占。这个路由顺序是正确的。

---

## 4. 本轮确认的主线问题

### 问题 1：代理端创建下级代理的 `max_sub_agents` 判断顺序有实质性 bug

**状态：已确认，P0 / P1 边界。**

`create_agent_in_scope_endpoint()` 当前流程是：

```text
1. 先创建 child Agent
2. db.add(child)
3. await db.flush()
4. 再读取当前代理 business_profile
5. 再判断 can_create_sub_agents
6. 再统计 existing_subs
7. if existing_subs >= max_sub: 拒绝
```

关键问题是：**统计 existing_subs 时，新 child 已经 flush 进当前事务。**

所以：

```text
max_sub_agents = 1
当前已有直属下级 = 0
创建第一个下级时：
  child 已 flush
  existing_subs = 1
  existing_subs >= max_sub_agents 成立
  请求被拒绝
```

这意味着当前逻辑会导致“刚好达到上限”的合法创建被拒绝。实际效果是：

```text
允许数量变成 max_sub_agents - 1
```

**建议方向：**

创建 child 前先判断：

```text
can_create_sub_agents
max_sub_agents
child_tier_level
username 唯一性
```

然后再 `db.add(child)`。

或者至少把判断改为“创建前计数”，不要把新 child 计入 existing_subs。

---

### 问题 2：代理端创建下级代理在权限检查前先写入 child，顺序不合理

**状态：已确认，P1。**

同一个函数中，当前先创建并 flush child，再判断当前代理是否允许创建下级。

虽然 FastAPI 的 `get_main_db()` 在异常时会 rollback，正常情况下不会永久落库，但这个顺序仍然有问题：

```text
权限检查失败依赖事务回滚兜底
而不是在业务前置条件失败前不写库
```

这类写法不利于长期维护，也会让审计、调试、事务边界变复杂。

**建议方向：**

前置检查顺序应为：

```text
1. username 唯一性
2. 当前代理业务画像
3. can_create_sub_agents
4. max_sub_agents
5. child_tier_level
6. 通过后再创建 Agent 和 AgentBusinessProfile
```

---

### 问题 3：测试 `test_agents_recursive.py` 已经落后于当前 `hierarchy_depth` 契约

**状态：已确认，P1。**

当前 `AgentTreeNode` schema 返回字段是：

```text
hierarchy_depth
```

但 `tests/test_agents_recursive.py` 中仍断言：

```python
data["root"]["level"] == 1
root_node["level"] == 1
child1_node["level"] == 2
gc["level"] == 3
```

当前 schema 中没有 `level` 字段。

**结论：**

这些测试如果运行，很可能失败。
这也说明“代理树测试存在，但没有跟上字段契约收敛”。

**建议方向：**

测试统一改成：

```python
data["root"]["hierarchy_depth"] == 1
```

不建议为了兼容测试重新返回 `level`，因为文档已经明确“不兼容旧字段 level / hierarchy_level”。

---

### 问题 4：代理端 Refresh 测试旧契约问题仍混在代理树测试文件里

**状态：已确认，延续上一轮认证审查发现。**

`tests/test_agents_recursive.py` 不只是代理树测试，还包含 `TestRevokeAll`，其中 refresh 请求仍只传：

```json
{
  "refresh_token": "..."
}
```

但当前 `RefreshRequest` 已要求：

```text
refresh_token
device_fingerprint
client_type
```

这说明该测试文件承担了两个主题：

```text
代理树测试
Token revoke-all 测试
```

而且其中 Token 部分已经落后。

**建议方向：**

把 `TestRevokeAll` 拆到认证专项测试中，例如：

```text
tests/test_auth_token_version.py
```

代理树文件只保留代理树与范围查询。

---

### 问题 5：Admin 创建代理与业务画像更新不是原子操作

**状态：已确认，P2。**

前端 `AgentCreateDialog.vue` 中，管理员创建代理时流程是：

```text
1. POST /api/agents/
2. 如果创建成功，再 PATCH /admin/api/agents/{id}/business-profile
```

后端 `POST /api/agents/` 只创建 Agent 主体，不同时创建业务画像。

如果第二步失败，会留下：

```text
Agent 已创建
AgentBusinessProfile 未按前端选择写入
后续读取时可能被默认创建为 Lv.1 / normal
```

这不是立即越权漏洞，但属于一致性风险。

**建议方向：**

长期更好方案有两个：

```text
方案 A：Admin 创建代理接口支持同时传入业务画像字段。
方案 B：新增专门的“创建代理 + 初始化画像”的服务层事务。
```

当前二段式前端调用可以继续用，但要认识到它不是原子流程。

---

### 问题 6：代理端给下级开通项目时，父级授权未明确拒绝“已过期 active 授权”

**状态：已确认，P1。**

`agent_scope_management.py` 中 `_active_project_auth()` 只检查：

```text
AgentProjectAuth.status == "active"
```

然后 `_ensure_parent_project_scope()` 才比较：

```text
父级 valid_until
下级 child_valid_until
```

但没有明确判断：

```text
父级 valid_until 已经过期 → 直接拒绝
```

如果父级授权状态仍是 active 但已经过期，逻辑上不应再允许给下级开通或更新授权。

虽然当前比较逻辑会阻止设置晚于父级到期时间的下级授权，但它没有清晰表达“父级授权已过期不可再下发”的业务规则。

**建议方向：**

在 `_ensure_parent_project_scope()` 中增加：

```text
parent_until is not None and parent_until <= now → 403
```

---

### 问题 7：代理范围管理允许操作整个子树，不只是直属下级，需明确是否符合设计

**状态：待确认 / 需产品口径确认。**

当前 `_target_agent_or_404()` 使用：

```text
get_agent_scope_ids(current_agent.id)
```

这表示当前代理可以操作：

```text
自己全部子树内的任意下级代理
```

包括孙级、曾孙级。

这与代码注释中的“下级代理”一致，但和某些业务语义中的“直属下级”可能不同。

例如代理端可以：

```text
查看孙级业务画像
修改孙级业务画像
给孙级开项目
给孙级划拨点数
重置孙级密码
```

**当前不能直接判定为 bug。**

需要确认业务口径到底是：

```text
A. 代理可管理整个子树
B. 代理只能管理直属下级
C. 查看可看整个子树，写操作只能直属下级
```

从当前源码看，采用的是 **A：可管理整个子树**。

如果项目长期治理希望更稳，建议至少在文档中明确这个设计，避免后续误解。

---

### 问题 8：代理端创建下级时使用业务等级低一级，但未检查 risk_status

**状态：已确认，P1，和上一轮项目准入发现一致。**

创建下级代理时当前检查：

```text
can_create_sub_agents
max_sub_agents
child_tier_level = parent_tier_level - 1
```

但没有检查当前代理自己的：

```text
risk_status
```

例如：

```text
risk_status = restricted
risk_status = frozen
```

是否还能创建下级代理？当前源码没有拦截。

**结论：**

这和上一轮“risk_status 尚未进入项目准入主链路”的问题一致。
代理树治理中也没有把风控状态作为硬约束。

---

### 问题 9：`get_agent_scope_ids()` 没有状态过滤

**状态：已确认，P2。**

`get_agent_scope_ids()` 递归查询：

```sql
WITH RECURSIVE scope AS (
  SELECT id FROM agent WHERE id = :agent_id
  UNION ALL
  SELECT a.id FROM agent a
  INNER JOIN scope s ON a.parent_agent_id = s.id
)
SELECT id FROM scope
```

没有过滤：

```text
Agent.status == active
```

当前含义是：

```text
权限范围 = 组织树结构范围
不是 active 业务范围
```

这不一定错，但需要统一口径。

当前结果是：

```text
代理范围列表可能包含 suspended 下级
代理端可能尝试查看/操作 suspended 下级
```

部分操作可能仍会成功，因为 `_target_agent_or_404()` 只校验是否在子树内，不校验目标代理是否 active。

**建议方向：**

需要明确：

```text
suspended 下级是否可被上级继续管理？
哪些操作允许？
哪些操作禁止？
```

如果 suspended 表示“停用代理业务能力”，则写操作应增加状态限制。

---

### 问题 10：代理管理测试覆盖不足

**状态：已确认，P1。**

当前 `tests/test_agents_recursive.py` 主要覆盖：

```text
Admin 子树查询
Admin 全树查询
Agent scope/list 权限范围查询
revoke-all 部分认证测试
```

但没有覆盖：

```text
代理创建下级
max_sub_agents 边界
can_create_sub_agents=false
child_tier_level 低一级
risk_status=frozen 是否禁止创建
代理更新下级基础信息
代理不能操作自己
代理不能操作平级
代理不能操作上级
代理给下级开项目
代理给下级划拨点数
代理冻结/解冻下级授信
代理重置下级密码
```

**结论：**

代理树查询有基础测试，但代理管理闭环没有形成门禁。

---

## 5. 当前代理管理大局观

### 已确认的真实状态

```text
已完成：
  Agent 自引用代理树模型
  hierarchy_depth 与 tier_level 概念区分
  WITH RECURSIVE 子树查询
  Admin 全代理列表 / 详情 / 子树 / 全树
  Agent scope/list 范围列表
  Agent scope/{id} 查看范围内代理
  Agent 创建直属下级代理
  Agent 更新范围内下级代理基础信息
  Agent 范围管理下级业务画像、项目授权、点数、密码
  前端 Admin / Agent 共用 AgentCreateDialog
  前端根据角色切换 /api/agents 和 /api/agents/scope
```

### 未闭合部分

```text
未闭合：
  代理端创建下级 max_sub_agents 判断顺序错误
  创建下级权限检查晚于写入
  risk_status 未作为代理管理硬约束
  父级项目授权过期未明确拒绝继续下发
  scope 写操作到底是整个子树还是直属下级，文档需定稿
  suspended 下级是否可继续被操作，规则不清
  Admin 创建代理 + 业务画像初始化不是原子事务
  代理管理测试落后且覆盖不足
```

---

## 6. 风险点汇总

| 风险                                                | 状态  |   严重度 |
| ------------------------------------------------- | --- | ----: |
| `max_sub_agents` 计数在 child flush 后执行，导致合法边界被拒绝    | 已确认 | P0/P1 |
| 创建下级代理权限检查晚于写入                                    | 已确认 |    P1 |
| 代理树测试仍断言旧字段 `level`                               | 已确认 |    P1 |
| `tests/test_agents_recursive.py` 混入旧 refresh 请求契约 | 已确认 |    P1 |
| 父级项目授权过期时未明确禁止继续给下级授权                             | 已确认 |    P1 |
| `risk_status` 未进入代理创建/管理硬约束                       | 已确认 |    P1 |
| scope 写操作覆盖整个子树，是否符合业务待确认                         | 待确认 |    P1 |
| suspended 下级是否允许被继续操作未明确                          | 待确认 |    P2 |
| Admin 创建代理与业务画像初始化非原子                             | 已确认 |    P2 |
| 代理管理专项测试不足                                        | 已确认 |    P1 |

---

## 7. 建议方案

### 第一优先级：修代理端创建下级代理顺序

建议把 `create_agent_in_scope_endpoint()` 改成：

```text
1. 校验 username 唯一
2. 获取 current_profile
3. 校验 can_create_sub_agents
4. 创建前统计直属下级数
5. 校验 max_sub_agents
6. 计算 child_tier_level
7. 校验 child_tier_level >= 1
8. 再创建 Agent
9. 再创建 AgentBusinessProfile
10. 写 audit_log
```

这是本轮最值得优先修的代理管理问题。

---

### 第二优先级：修测试契约

优先修：

```text
tests/test_agents_recursive.py
```

两件事：

```text
1. level → hierarchy_depth
2. 把 TestRevokeAll 拆出去，或者至少补齐 refresh 请求体
```

否则代理树测试当前不能作为可信质量门禁。

---

### 第三优先级：明确 scope 写权限范围

建议在文档里明确：

```text
代理端 scope 写操作是否允许操作整个子树？
```

如果允许，写清楚：

```text
上级代理可管理所有后代代理，不限直属下级。
```

如果不允许，则 `_target_agent_or_404()` 需要增加：

```text
target.parent_agent_id == current_agent.id
```

用于写操作；查看操作可以保留整棵子树。

---

### 第四优先级：把 risk_status 纳入代理管理硬约束

建议至少先定义：

```text
normal：正常
watch：允许操作但可标记审查
restricted：禁止创建下级 / 禁止自动开通 / 禁止给下级授权
frozen：禁止所有写操作，仅允许查看
```

然后在以下入口统一接入：

```text
create_agent_in_scope_endpoint
scope_update_business_profile
scope_grant_project_auth
scope_recharge / credit / freeze / unfreeze
scope_reset_password
```

---

### 第五优先级：补代理管理专项测试

建议新增或拆分：

```text
tests/test_agent_scope_management.py
tests/test_agent_create_subagent.py
tests/test_agent_tree_permissions.py
```

第一批必须覆盖：

```text
1. can_create_sub_agents=false 不能创建下级
2. max_sub_agents=1 时可创建第一个、不能创建第二个
3. Lv.1 不能继续创建下级
4. Lv.3 创建下级自动为 Lv.2
5. 代理不能操作自己
6. 代理不能操作平级
7. 代理不能操作上级
8. scope/list 只返回自己子树
9. frozen 代理不能创建下级
10. suspended 下级写操作按最终规则验证
```

---

## 8. 当前结论

**已确认：**

代理树基础模型、递归查询、Admin / Agent 路由边界已经成型。

**已确认：**

`hierarchy_depth` 与 `tier_level` 的概念区分在主要代码中基本成立。

**已确认：**

代理端范围管理已经非常接近完整业务模块，不是空白实现。

**已确认：**

当前最严重的代理管理问题是：**代理端创建下级代理的顺序和计数逻辑有实质性错误**，会导致 `max_sub_agents` 边界判断异常。

**待确认：**

代理 scope 写操作到底应该覆盖整棵子树还是只限直属下级，这需要你定业务口径。

---

## 9. 推荐下一步

下一轮建议继续主线，进入 **用户管理与授权开通治理**：

```text
1. app/routers/users.py
2. app/services/user_service.py 复读
3. app/schemas/user.py
4. 授权升级续费与均分方案.md
5. 代理点数计费与返点规则.md
6. tests/test_authorization_upgrade.py
7. tests/test_accounting.py 中授权扣点部分
```

目标是确认：

```text
用户创建归属是否清楚；
Admin / Agent 操作用户的边界是否清楚；
代理是否只能操作自己创建的用户；
项目授权、升级、续费、设备数、扣点、返点是否一致；
用户授权是否正确依赖代理项目授权和账务余额。
```
