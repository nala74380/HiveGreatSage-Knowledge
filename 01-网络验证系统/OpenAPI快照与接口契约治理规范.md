---
文件位置: 01-网络验证系统/OpenAPI快照与接口契约治理规范.md
名称: Verify OpenAPI 快照与接口契约治理规范
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-29
版本: V1.0.0
状态: 草稿
关联文档:
  - "[[项目总大纲]]"
  - "[[01-网络验证系统/架构设计]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/前后端接口对照表]]"
  - "[[01-网络验证系统/三端联调测试清单]]"
  - "[[01-网络验证系统/风险与待决策清单]]"
  - "[[01-网络验证系统/仓库治理与敏感配置清理清单]]"
  - "[[02-PC中控框架/Verify接口调用清单]]"
  - "[[03-安卓脚本框架/Verify接口调用清单]]"
  - "[[编码规范]]"
  - "[[编码规范_补充_Markdown文档规范]]"
变更记录:
  - V1.0.0: 基于 HiveGreatSage-Verify 当前 app/main.py 路由注册与 FastAPI OpenAPI 能力，整理 OpenAPI 快照生成、归档、比对、破坏性变更治理与三端契约同步规范
---

# Verify OpenAPI 快照与接口契约治理规范

← 返回 [[项目总大纲]] | 父节点: [[01-网络验证系统/API路由清单]]

## 一、文档目的

本文用于规范 `HiveGreatSage-Verify` 的 **OpenAPI 快照生成、归档、比对和接口契约治理**。

本文解决的问题是：

```text
1. 当前 Verify 真实暴露了哪些接口？
2. PCControl、AndroidScript、Web 前端应该以哪个接口契约为准？
3. API 路由清单和真实源码如何防止漂移？
4. 接口字段变更如何被发现？
5. 路由路径变更如何同步给三端？
6. 哪些变更属于破坏性变更？
7. 每个阶段如何生成可追溯的接口快照？
8. OpenAPI 快照如何进入 Obsidian 知识真相源？
```

---

## 二、当前结论

### 2.1 已确认

当前 Verify 使用 FastAPI。

FastAPI 应用入口为：

```text
app/main.py
```

当前应用实例：

```python
app = FastAPI(
    title="HiveGreatSage-Verify",
    description="蜂巢·大圣平台 — 网络验证系统（中枢）",
    version="0.1.0",
    docs_url="/docs" if settings.DEBUG else None,
    redoc_url="/redoc" if settings.DEBUG else None,
    lifespan=lifespan,
)
```

已确认：

```text
1. DEBUG=true 时开放 /docs 和 /redoc。
2. DEBUG=false 时关闭 /docs 和 /redoc。
3. /health 始终作为健康检查接口存在。
4. 路由集中在 app/main.py 注册。
5. OpenAPI Schema 可通过 app.openapi() 生成。
```

---

### 2.2 当前已注册路由模块

当前 `app/main.py` 已注册：

```text
/api/auth
/api/users
/api/agents
/api/agents/my/project-access
/api/device
/api/params
/api/update
/admin/api
/admin/api/projects
/admin/api/updates
/admin/api/devices
/api/stats
/admin/api/project-access
/admin/api
```

对应模块：

```text
auth
users
balance_agent
project_access_agent
agents
device
params
update
admin
projects
update_admin
device_admin
stats
balance_admin
project_access_admin
agent_profile_admin
```

---

### 2.3 当前路由注册特殊风险

当前源码已明确：

```text
balance_agent 必须在 agents.router 之前注册。
```

原因：

```text
agents.router 内可能存在 /{agent_id} 动态路由。
如果 agents.router 先注册，则 /api/agents/catalog 可能被误匹配为 /api/agents/{agent_id}。
```

这说明：

```text
OpenAPI 快照不仅要记录接口路径，还要辅助发现路由顺序风险。
```

---

### 2.4 当前状态判断

当前状态：

```text
源码已集中注册路由。
API 路由清单已有人工整理。
前后端接口对照表已有反向整理。
但尚未建立“每阶段强制生成 OpenAPI 快照”的治理制度。
```

风险等级：

```text
P2 中风险。
```

原因：

```text
1. 接口数量已经明显增加。
2. Web 前端、PCControl、AndroidScript 都依赖 Verify。
3. 手写 API 文档容易落后于源码。
4. 路由路径、字段、响应结构变化后，三端可能不同步。
```

---

## 三、OpenAPI 快照定位

## 3.1 OpenAPI 快照是什么

OpenAPI 快照是从 FastAPI 应用当前源码生成的接口契约 JSON。

它记录：

```text
1. 路径。
2. HTTP 方法。
3. Tag。
4. Query 参数。
5. Path 参数。
6. Request Body Schema。
7. Response Schema。
8. 错误响应。
9. Pydantic Schema。
10. 安全认证声明。
```

---

## 3.2 OpenAPI 快照不是什么

OpenAPI 快照不是：

```text
1. 架构设计文档。
2. 业务规则文档。
3. 三端联调报告。
4. 权限测试结果。
5. 数据库设计文档。
6. 前端接口对照表的替代品。
```

OpenAPI 只能说明：

```text
源码当前声明了什么接口契约。
```

不能证明：

```text
1. 接口业务逻辑正确。
2. 权限边界正确。
3. 数据库事务正确。
4. 三端调用已经通过。
5. 热更新安装已经成功。
```

---

## 3.3 OpenAPI 快照的项目价值

对 Verify：

```text
防止后端路由变化后文档不同步。
```

对 Web 前端：

```text
确认管理后台、代理后台调用路径是否存在。
```

对 PCControl：

```text
确认 /api/auth、/api/device、/api/params、/api/update 的字段契约。
```

对 AndroidScript：

```text
确认登录、心跳、参数、热更新接口字段是否与 Lua 代码一致。
```

对 Obsidian：

```text
沉淀每个阶段真实接口状态，避免靠记忆推进项目。
```

---

## 四、快照文件命名规范

## 4.1 JSON 原始快照

建议保存到：

```text
docs/openapi/openapi_YYYY-MM-DD.json
```

示例：

```text
docs/openapi/openapi_2026-04-29.json
```

如果仓库不希望保存生成产物，可保存到：

```text
artifacts/openapi/openapi_2026-04-29.json
```

并在 Obsidian 记录生成结果。

---

## 4.2 Markdown 摘要

建议复制进 Obsidian：

```text
01-网络验证系统/OpenAPI快照_YYYY-MM-DD.md
```

示例：

```text
01-网络验证系统/OpenAPI快照_2026-04-29.md
```

---

## 4.3 差异报告

当接口发生变化时生成：

```text
01-网络验证系统/OpenAPI差异报告_YYYY-MM-DD.md
```

示例：

```text
01-网络验证系统/OpenAPI差异报告_2026-04-29.md
```

---

## 五、快照生成命令

## 5.1 直接生成 JSON

在 Verify 仓库根目录执行：

```bash
python - <<'PY'
import json
from pathlib import Path
from app.main import app

out_dir = Path("docs/openapi")
out_dir.mkdir(parents=True, exist_ok=True)

schema = app.openapi()
out_file = out_dir / "openapi_2026-04-29.json"

with out_file.open("w", encoding="utf-8") as f:
    json.dump(schema, f, ensure_ascii=False, indent=2)

print("OpenAPI title:", schema.get("info", {}).get("title"))
print("OpenAPI version:", schema.get("info", {}).get("version"))
print("paths:", len(schema.get("paths", {})))
print("schemas:", len(schema.get("components", {}).get("schemas", {})))
print("output:", out_file)
PY
```

---

## 5.2 生成路径摘要

```bash
python - <<'PY'
from app.main import app

schema = app.openapi()
paths = schema.get("paths", {})

for path, methods in sorted(paths.items()):
    for method in sorted(methods.keys()):
        if method.lower() in {"get", "post", "put", "patch", "delete"}:
            item = methods[method]
            tags = ",".join(item.get("tags", []))
            summary = item.get("summary", "")
            print(f"{method.upper():6} {path:60} [{tags}] {summary}")
PY
```

---

## 5.3 生成 Markdown 路由表

```bash
python - <<'PY'
from pathlib import Path
from app.main import app

schema = app.openapi()
paths = schema.get("paths", {})

lines = []
lines.append("| HTTP | Path | Tags | Summary |")
lines.append("|---|---|---|---|")

for path, methods in sorted(paths.items()):
    for method in sorted(methods.keys()):
        if method.lower() not in {"get", "post", "put", "patch", "delete"}:
            continue
        item = methods[method]
        tags = ", ".join(item.get("tags", []))
        summary = item.get("summary", "") or ""
        lines.append(f"| {method.upper()} | `{path}` | {tags} | {summary} |")

out = Path("docs/openapi/openapi_routes_2026-04-29.md")
out.parent.mkdir(parents=True, exist_ok=True)
out.write_text("\n".join(lines), encoding="utf-8")

print("output:", out)
PY
```

---

## 六、快照摘要模板

复制到 Obsidian：

```markdown
---
文件位置: 01-网络验证系统/OpenAPI快照_YYYY-MM-DD.md
名称: Verify OpenAPI 快照 YYYY-MM-DD
作者: 蜂巢·大圣 (HiveGreatSage)
时间: YYYY-MM-DD
版本: V1.0.0
状态: 草稿
关联文档:
  - "[[01-网络验证系统/OpenAPI快照与接口契约治理规范]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/前后端接口对照表]]"
变更记录:
  - V1.0.0: 本次 OpenAPI 快照摘要
---

# Verify OpenAPI 快照 YYYY-MM-DD

## 一、生成环境

```text
Verify commit:
生成时间:
Python:
FastAPI:
ENVIRONMENT:
DEBUG:
```

## 二、快照统计

```text
OpenAPI title:
OpenAPI version:
paths 数量:
schemas 数量:
```

## 三、核心路径摘要

| HTTP | Path | Tags | Summary |
|---|---|---|---|
|  |  |  |  |

## 四、与上一次快照差异

```text
新增接口:
1.

删除接口:
1.

修改接口:
1.

待确认:
1.
```

## 五、影响范围

```text
Web 前端:
PCControl:
AndroidScript:
测试:
文档:
```

## 六、结论

```text
□ 可作为当前阶段接口契约
□ 有破坏性变化，需同步三端
□ 快照生成失败
□ 待补充
```
```

---

## 七、接口分组治理

## 7.1 用户态接口

用户态接口由 PCControl / AndroidScript 使用。

当前主要前缀：

```text
/api/auth
/api/device
/api/params
/api/update
```

必须重点关注：

```text
1. 登录字段。
2. Token 刷新字段。
3. 心跳字段。
4. 设备列表字段。
5. 参数 GET/SET 字段。
6. 热更新 check/download 字段。
```

任何用户态接口变更，都必须同步：

```text
[[02-PC中控框架/Verify接口调用清单]]
[[03-安卓脚本框架/Verify接口调用清单]]
[[01-网络验证系统/三端联调测试清单]]
```

---

## 7.2 管理员后台接口

管理员后台接口主要前缀：

```text
/admin/api
/admin/api/projects
/admin/api/updates
/admin/api/devices
/admin/api/project-access
```

必须重点关注：

```text
1. 项目管理。
2. 用户管理附加操作。
3. 热更新上传。
4. 设备监控。
5. 点数管理。
6. 项目准入策略。
7. 代理业务画像。
```

任何管理员接口变更，都必须同步：

```text
[[01-网络验证系统/前后端接口对照表]]
[[01-网络验证系统/前端管理后台架构]]
```

---

## 7.3 代理后台接口

代理后台接口主要前缀：

```text
/api/agents
/api/agents/my/project-access
```

必须重点关注：

```text
1. 代理登录。
2. 代理余额。
3. 代理流水。
4. 项目价格目录。
5. 项目准入目录。
6. 项目申请。
7. 用户管理共用接口。
```

任何代理接口变更，都必须同步：

```text
[[01-网络验证系统/前后端接口对照表]]
[[01-网络验证系统/项目准入策略与代理开通规则]]
[[01-网络验证系统/代理点数计费与返点规则]]
```

---

## 八、破坏性变更定义

以下都属于破坏性变更：

```text
1. 删除接口。
2. 修改接口路径。
3. 修改 HTTP 方法。
4. 修改必填字段。
5. 删除响应字段。
6. 修改字段类型。
7. 修改枚举值。
8. 修改错误码语义。
9. 修改鉴权方式。
10. 修改 Token 所属身份。
11. 修改上传字段名。
12. 修改分页结构。
```

---

## 九、非破坏性变更定义

以下通常属于非破坏性变更：

```text
1. 新增可选响应字段。
2. 新增可选请求字段。
3. 新增接口。
4. 新增说明文字。
5. 新增非必填 Query 参数。
```

但仍需要同步文档。

原因：

```text
非破坏性不等于不需要记录。
```

---

## 十、接口变更流程

## 10.1 修改前

必须先回答：

```text
1. 为什么要改？
2. 影响哪个端？
3. 是否有旧客户端仍在使用？
4. 是否需要兼容期？
5. 是否需要版本化接口？
6. 是否需要修改测试？
7. 是否需要更新 Obsidian？
```

---

## 10.2 修改中

必须同步修改：

```text
1. Router。
2. Schema。
3. Service。
4. Tests。
5. Frontend API。
6. PCControl API Client。
7. AndroidScript Lua 调用。
```

视影响范围而定，不是每次都全部修改。

---

## 10.3 修改后

必须执行：

```text
1. 生成 OpenAPI 快照。
2. 与上一次快照对比。
3. 跑相关 pytest。
4. 跑 frontend build。
5. 如果影响 PC/安卓，跑三端联调。
6. 更新 Obsidian 文档。
```

---

## 十一、OpenAPI 差异比对

## 11.1 简单路径差异脚本

```bash
python - <<'PY'
import json
from pathlib import Path

old_file = Path("docs/openapi/openapi_old.json")
new_file = Path("docs/openapi/openapi_new.json")

old = json.loads(old_file.read_text(encoding="utf-8"))
new = json.loads(new_file.read_text(encoding="utf-8"))

old_paths = set(old.get("paths", {}).keys())
new_paths = set(new.get("paths", {}).keys())

print("新增路径:")
for p in sorted(new_paths - old_paths):
    print(" +", p)

print("删除路径:")
for p in sorted(old_paths - new_paths):
    print(" -", p)

print("共同路径数量:", len(old_paths & new_paths))
PY
```

---

## 11.2 方法级差异脚本

```bash
python - <<'PY'
import json
from pathlib import Path

old = json.loads(Path("docs/openapi/openapi_old.json").read_text(encoding="utf-8"))
new = json.loads(Path("docs/openapi/openapi_new.json").read_text(encoding="utf-8"))

def operations(schema):
    result = set()
    for path, methods in schema.get("paths", {}).items():
        for method in methods.keys():
            if method.lower() in {"get", "post", "put", "patch", "delete"}:
                result.add((method.upper(), path))
    return result

old_ops = operations(old)
new_ops = operations(new)

print("新增操作:")
for method, path in sorted(new_ops - old_ops):
    print(f" + {method} {path}")

print("删除操作:")
for method, path in sorted(old_ops - new_ops):
    print(f" - {method} {path}")
PY
```

---

## 十二、必须纳入快照治理的高风险接口

## 12.1 登录接口

```text
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout
POST /api/auth/revoke-all
GET  /api/auth/me
```

高风险原因：

```text
1. PCControl 和 AndroidScript 都依赖。
2. Token 字段变化会影响全部客户端。
3. revoke-all 后续会引入 token_version。
```

---

## 12.2 设备接口

```text
POST /api/device/heartbeat
GET  /api/device/list
GET  /api/device/data
POST /api/device/imsi
```

高风险原因：

```text
1. AndroidScript 心跳依赖。
2. PCControl 设备列表依赖。
3. 设备绑定项目维度迁移会影响字段。
4. Redis 心跳落库链路依赖该契约。
```

---

## 12.3 参数接口

```text
GET  /api/params/get
POST /api/params/set
```

高风险原因：

```text
1. PCControl 保存参数。
2. AndroidScript 拉取参数。
3. 参数值格式 JSON 字符串 / 原生类型必须统一。
```

---

## 12.4 热更新接口

```text
GET  /api/update/check
GET  /api/update/download
POST /admin/api/updates/{project_id}/{client_type}
GET  /admin/api/updates/{project_id}/{client_type}/latest
GET  /admin/api/updates/{project_id}/{client_type}/history
```

高风险原因：

```text
1. 版本字段 version / latest_version 曾存在漂移风险。
2. project_id / game_project_code 路径参数曾存在漂移风险。
3. 主库 / 游戏库 version_record 真相源曾存在漂移风险。
```

---

## 12.5 点数接口

```text
GET  /admin/api/balance-transactions
GET  /admin/api/prices/{project_id}
PUT  /admin/api/prices/{project_id}/{user_level}
DELETE /admin/api/prices/{project_id}/{user_level}
POST /admin/api/agents/{agent_id}/recharge
POST /admin/api/agents/{agent_id}/credit
POST /admin/api/agents/{agent_id}/freeze
POST /admin/api/agents/{agent_id}/unfreeze
GET  /api/agents/my/balance
GET  /api/agents/my/transactions
```

高风险原因：

```text
1. 涉及代理财务。
2. 字段 points_per_device / price_points 曾有漂移风险。
3. 字段 description / note 曾有漂移风险。
```

---

## 12.6 项目准入接口

```text
GET   /api/agents/my/project-access/catalog
POST  /api/agents/my/project-access/requests
GET   /api/agents/my/project-access/requests
POST  /api/agents/my/project-access/requests/{request_id}/cancel
GET   /admin/api/project-access/policies
PATCH /admin/api/project-access/policies/{project_id}
GET   /admin/api/project-access/requests
POST  /admin/api/project-access/requests/{request_id}/approve
POST  /admin/api/project-access/requests/{request_id}/reject
```

高风险原因：

```text
1. 决定代理能否售卖项目。
2. access_status / action_type 必须与前端对齐。
3. tier_level 不能和 Agent.level 混用。
```

---

## 十三、OpenAPI 与手写文档的关系

## 13.1 OpenAPI 是机器快照

OpenAPI 适合记录：

```text
1. 路径。
2. 方法。
3. 参数。
4. Schema。
5. 认证。
```

---

## 13.2 手写文档是业务真相源

Obsidian 文档适合记录：

```text
1. 为什么这样设计。
2. 模块边界。
3. 风险。
4. 决策。
5. 测试场景。
6. 回退方案。
7. 多端协作口径。
```

---

## 13.3 两者必须互相校验

如果 OpenAPI 与手写文档冲突：

```text
1. 先以源码/OpenAPI 判断当前真实实现。
2. 再判断文档是否落后。
3. 如果源码实现不合理，不能盲目改文档迎合源码。
4. 应建立风险项或修订任务。
```

---

## 十四、CI 建议

后续 CI 应加入：

```text
1. 生成 OpenAPI。
2. 校验 JSON 格式。
3. 检查 paths 数量不为 0。
4. 可选：和基线快照比对。
5. 如果有删除接口，要求人工确认。
```

最小脚本：

```bash
python - <<'PY'
from app.main import app

schema = app.openapi()
paths = schema.get("paths", {})

if not paths:
    raise SystemExit("OpenAPI paths is empty")

print("OpenAPI paths:", len(paths))
PY
```

---

## 十五、当前风险清单

## R001：未建立阶段性 OpenAPI 快照

风险等级：

```text
P2
```

影响：

```text
API 文档可能继续靠人工记忆维护。
```

处理：

```text
每个阶段生成 openapi_YYYY-MM-DD.json 和 OpenAPI快照_YYYY-MM-DD.md。
```

---

## R002：路由顺序风险不容易被手写文档发现

风险等级：

```text
P2
```

已确认：

```text
balance_agent 必须在 agents.router 之前注册。
```

处理：

```text
OpenAPI 快照生成后，额外跑关键路径冒烟测试。
例如 /api/agents/catalog 不应被 /api/agents/{agent_id} 抢占。
```

---

## R003：生产环境关闭 /docs 后缺少离线文档

风险等级：

```text
P2
```

已确认：

```text
DEBUG=false 时 docs_url / redoc_url 为 None。
```

处理：

```text
生产前必须生成离线 OpenAPI 快照，并归档。
```

---

## R004：Schema 字段变更未自动通知三端

风险等级：

```text
P1
```

处理：

```text
OpenAPI 差异报告必须标记影响端：
Web 前端 / PCControl / AndroidScript。
```

---

## 十六、建议新增脚本

建议新增：

```text
scripts/export_openapi.py
```

职责：

```text
1. 导入 app.main:app。
2. 生成 openapi JSON。
3. 生成路由 Markdown 表。
4. 输出 paths / schemas 数量。
```

建议新增：

```text
scripts/diff_openapi.py
```

职责：

```text
1. 比较两个 OpenAPI JSON。
2. 输出新增/删除/修改路径。
3. 输出新增/删除 Schema。
4. 生成 Markdown 差异报告。
```

---

## 十七、建议新增决策日志

```markdown
## D038 — Verify 每个阶段必须生成 OpenAPI 快照

**日期：** 2026-04-29  
**状态：** 建议确认  
**关联文档：** [[01-网络验证系统/OpenAPI快照与接口契约治理规范]]

### 背景

Verify 已经成为 PCControl、AndroidScript、管理后台、代理后台共同依赖的平台中枢。接口数量持续增加，人工维护 API 清单容易漂移。

### 决策

每个阶段基线必须生成：

```text
docs/openapi/openapi_YYYY-MM-DD.json
01-网络验证系统/OpenAPI快照_YYYY-MM-DD.md
```

当接口发生变化时，生成：

```text
01-网络验证系统/OpenAPI差异报告_YYYY-MM-DD.md
```

### 理由

```text
1. 让接口契约可追溯。
2. 防止前后端路径漂移。
3. 防止 PCControl / AndroidScript 字段不一致。
4. 支撑三端联调。
5. 支撑生产环境关闭 /docs 后仍可查接口契约。
```

### 后续动作

```text
□ 新增 scripts/export_openapi.py。
□ 新增 scripts/diff_openapi.py。
□ 生成首次 openapi_2026-04-29.json。
□ 生成 OpenAPI快照_2026-04-29.md。
```
```

```markdown
## D039 — OpenAPI 快照不能替代业务规则文档

**日期：** 2026-04-29  
**状态：** 建议确认  
**关联文档：** [[01-网络验证系统/OpenAPI快照与接口契约治理规范]]

### 背景

OpenAPI 能反映接口路径和字段，但不能解释项目准入、点数返点、设备绑定、Token 吊销、热更新发布治理等业务规则。

### 决策

OpenAPI 只作为机器接口契约快照。

业务规则仍以 Obsidian 手写文档为真相源。

### 理由

```text
1. OpenAPI 不表达业务意图。
2. OpenAPI 不表达风险和回退。
3. OpenAPI 不表达多端协作边界。
4. OpenAPI 不表达财务规则。
5. OpenAPI 不表达长期架构决策。
```

### 后续动作

```text
□ OpenAPI 快照与 API路由清单互相校验。
□ OpenAPI 差异与前后端接口对照表互相校验。
□ 高风险业务规则继续维护独立文档。
```
```

---

## 十八、推荐下一步

```text
Step 1:
  把本文复制进 Obsidian:
  01-网络验证系统/OpenAPI快照与接口契约治理规范.md

Step 2:
  在 Verify 仓库新增:
  scripts/export_openapi.py

Step 3:
  生成:
  docs/openapi/openapi_2026-04-29.json
  docs/openapi/openapi_routes_2026-04-29.md

Step 4:
  复制摘要到 Obsidian:
  01-网络验证系统/OpenAPI快照_2026-04-29.md

Step 5:
  对照:
  [[01-网络验证系统/API路由清单]]
  [[01-网络验证系统/前后端接口对照表]]

Step 6:
  标记不一致接口。

Step 7:
  如果发现破坏性变更，生成:
  01-网络验证系统/OpenAPI差异报告_2026-04-29.md

Step 8:
  把 OpenAPI 生成纳入后续测试基线。
```

---

## 十九、结论

当前 Verify 已经具备通过 FastAPI 自动生成 OpenAPI 的基础。

已确认：

```text
1. app/main.py 集中创建 FastAPI app。
2. 当前 docs_url / redoc_url 只在 DEBUG 下开放。
3. 生产环境不会直接开放 /docs。
4. 路由注册集中在 app/main.py。
5. 当前路由已覆盖 Verify 核心平台能力。
6. 路由注册顺序已经出现需要治理的真实风险。
```

当前最准确状态：

```text
OpenAPI 生成能力已具备；
但阶段性快照治理尚未制度化。
```

下一步重点：

```text
不是继续手写更多散落接口说明；
而是生成首次 OpenAPI 快照，
再用机器快照反向校验 API路由清单、前后端接口对照表、PCControl 调用清单和 AndroidScript 调用清单。
```