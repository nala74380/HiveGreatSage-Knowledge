---
文件位置: 01-网络验证系统/Redis心跳落库策略.md
名称: Redis 心跳落库策略
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-29
版本: V1.1.0
状态: 草稿
关联文档:
  - "[[项目总大纲]]"
  - "[[01-网络验证系统/架构设计]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/数据库设计]]"
  - "[[01-网络验证系统/三端联调测试清单]]"
  - "[[01-网络验证系统/测试基线_2026-04-29]]"
  - "[[01-网络验证系统/风险与待决策清单]]"
  - "[[02-PC中控框架/Verify接口调用清单]]"
  - "[[03-安卓脚本框架/Verify接口调用清单]]"
  - "[[编码规范]]"
  - "[[编码规范_补充_Markdown文档规范]]"
变更记录:
  - V1.1.0: 基于 HiveGreatSage-Verify 当前源码反向整理 Redis 心跳缓存、PC 查询、Celery 批量落库、限流、离线判定、风险与测试清单
  - V1.0.0: 初版 Redis 心跳落库策略
---

# Redis 心跳落库策略

← 返回 [[项目总大纲]] | 父节点: [[01-网络验证系统/架构设计]]

## 一、文档目的

本文用于沉淀 `HiveGreatSage-Verify` 中 **安卓设备心跳 → Redis 实时缓存 → Celery 批量落库 → PostgreSQL 游戏库 → PC 中控查询** 的完整策略。

本文重点回答：

```text
1. 为什么安卓心跳不直接写 PostgreSQL？
2. Redis 心跳 Key 如何设计？
3. 心跳写入后如何判定设备在线？
4. PC 中控如何获取在线和离线设备？
5. Celery 如何批量落库？
6. 游戏库 device_runtime 表如何更新？
7. 当前源码已经实现什么？
8. 当前还需要测试什么？
9. 未来 10,000+ 设备规模下有哪些风险？
```

---

## 二、当前结论

### 2.1 已确认

当前 Verify 源码已经实现 Redis 心跳缓冲与 Celery 批量落库链路。

已确认源码层面存在：

```text
1. POST /api/device/heartbeat
   安卓脚本心跳上报接口。

2. D5 限流
   同 user_id 每分钟最多 4 次。

3. Redis 心跳 Key
   device:runtime:{game_id}:{user_id}:{device_fp}

4. Redis 心跳 TTL
   120 秒。

5. PC 设备列表
   GET /api/device/list

6. 单设备详情
   GET /api/device/data

7. Celery 定时任务
   app.tasks.heartbeat_flush.flush_device_heartbeats

8. Celery 调度频率
   每 30 秒执行一次。

9. 游戏库落库表
   device_runtime

10. 批量 UPSERT
   ON CONFLICT (device_id) DO UPDATE。
```

### 2.2 待确认

源码存在不等于链路已经在当前环境跑通。

仍需确认：

```text
1. Redis 服务是否正常启动。
2. /api/device/heartbeat 是否能真实写入 Redis。
3. Celery Worker 是否正常启动。
4. Celery Beat 是否每 30 秒触发任务。
5. flush_device_heartbeats 是否能扫描到 Redis Key。
6. 心跳是否能写入正确的游戏库。
7. device_runtime 是否能正确 UPSERT。
8. PCControl 是否能看到 Redis 在线设备。
9. Redis 过期后 PCControl 是否能看到游戏库离线设备。
10. 大量设备并发上报时 Redis / Celery / PostgreSQL 是否稳定。
```

### 2.3 当前状态口径

```text
源码已实现:
  心跳写 Redis、PC 查询 Redis + 游戏库、Celery 批量落库。

待运行验证:
  Redis / Celery / PostgreSQL 三件套是否在当前环境完整跑通。

待三端联调:
  AndroidScript 上报心跳后，PCControl 是否能看到设备。

待压测:
  1,000 / 5,000 / 10,000 台设备模拟心跳。
```

---

## 三、为什么心跳不直接写 PostgreSQL

### 3.1 规模假设

项目总规模目标：

```text
单个游戏项目:
  10,000+ 台安卓设备。

单个 VIP/SVIP 客户:
  可能有 3,000+ 台设备。

心跳频率:
  每台设备每 30 秒上报一次。

压力估算:
  10,000 / 30 ≈ 333 次/秒。
```

如果每次心跳都直接写 PostgreSQL：

```text
1. PostgreSQL 持续高频 UPDATE。
2. 连接池压力变大。
3. I/O 压力变大。
4. device_runtime 行锁竞争增加。
5. API 响应时间受数据库抖动影响。
6. PC 查询、后台查询、授权操作都可能被心跳写入拖慢。
```

### 3.2 当前方案

当前采用：

```text
AndroidScript
  → Verify API
  → Redis 实时缓存
  → Celery 每 30 秒批量 UPSERT
  → PostgreSQL 游戏库 device_runtime
```

### 3.3 方案收益

```text
1. 心跳接口响应快。
2. Redis 承受高频写入。
3. PostgreSQL 只承受批量落库。
4. PC 中控能读到实时在线状态。
5. 离线设备仍可从游戏库读取历史数据。
6. Celery 单项目失败不会影响其他项目继续落库。
```

---

## 四、总体数据流

```text
┌──────────────────────┐
│    AndroidScript      │
│  懒人精灵安卓脚本       │
└──────────┬───────────┘
           │
           │ POST /api/device/heartbeat
           │ Authorization: Bearer UserToken
           │
           ▼
┌──────────────────────┐
│       Verify API      │
│ app/routers/device.py │
└──────────┬───────────┘
           │
           │ 1. 校验 User Token
           │ 2. 解析 game_project_code
           │ 3. D5 限流
           │ 4. 校验设备已绑定
           │
           ▼
┌──────────────────────────────┐
│    DeviceService              │
│ app/services/device_service.py │
└──────────┬───────────────────┘
           │
           │ set_heartbeat()
           │
           ▼
┌─────────────────────────────────────────────────┐
│                    Redis                         │
│ device:runtime:{game_id}:{user_id}:{device_fp}   │
│ TTL = 120 秒                                     │
└──────────┬──────────────────────────────────────┘
           │
           │ PC 查询实时状态
           │
           ├───────────────────────────────┐
           │                               │
           ▼                               ▼
┌──────────────────────┐        ┌──────────────────────────┐
│      PCControl        │        │          Celery           │
│ GET /api/device/list  │        │ flush_device_heartbeats   │
└──────────────────────┘        └────────────┬─────────────┘
                                             │
                                             │ 每 30 秒
                                             │ 扫描 device:runtime:{game_id}:*
                                             │ 批量 UPSERT
                                             ▼
                              ┌──────────────────────────────┐
                              │ PostgreSQL 游戏库             │
                              │ hive_game_xxx.device_runtime  │
                              └──────────────────────────────┘
```

---

## 五、接口层策略

## 5.1 安卓心跳接口

接口：

```text
POST /api/device/heartbeat
```

调用方：

```text
AndroidScript
```

鉴权：

```text
User Token
```

请求体：

```json
{
  "device_fingerprint": "android_device_001",
  "status": "running",
  "game_data": {
    "current_task": "daily",
    "map": "东海岸",
    "gold": 1024
  }
}
```

响应：

```json
{
  "code": 0,
  "message": "ok"
}
```

### 5.1.1 路由层职责

`app/routers/device.py` 只负责：

```text
1. 鉴权。
2. 取得当前用户。
3. 取得当前项目 code。
4. 心跳限流。
5. 调用 DeviceService.process_heartbeat。
6. 返回响应。
```

路由层不负责：

```text
1. 设备绑定业务判断。
2. Redis Key 拼接。
3. 数据库落库。
4. PC 查询逻辑。
```

### 5.1.2 限流规则

当前规则：

```text
同 user_id 每分钟最多 4 次。
```

配置：

```text
_HEARTBEAT_RATE_LIMIT = 4
_HEARTBEAT_RATE_WINDOW = 60
```

含义：

```text
默认安卓 30 秒心跳一次。
每分钟 2 次。
上限设置为 4 次，给重试留余量。
```

### 5.1.3 限流风险

如果 AndroidScript 心跳间隔小于 15 秒：

```text
可能触发 429 Too Many Requests。
```

建议：

```text
1. AndroidScript 默认 HEARTBEAT_INTERVAL = 30。
2. 最小值不低于 20 秒。
3. 收到 429 后应延长间隔，而不是立即重试。
```

---

## 六、服务层策略

## 6.1 process_heartbeat 流程

`process_heartbeat()` 当前执行：

```text
1. 通过 game_project_code 查询主库 game_project。
2. 校验设备已经绑定到当前用户。
3. 更新主库 DeviceBinding.last_seen_at。
4. 构造心跳 payload。
5. 调用 set_heartbeat 写 Redis。
6. 返回 HeartbeatResponse。
```

### 6.1.1 设备绑定校验

校验条件：

```text
DeviceBinding.user_id == current_user.id
DeviceBinding.device_fingerprint == body.device_fingerprint
DeviceBinding.status == "active"
```

如果设备未绑定：

```text
HTTP 403
detail = "设备未绑定到当前账号，拒绝上报"
```

### 6.1.2 为什么必须校验设备绑定

防止：

```text
1. 用户伪造其他设备指纹。
2. 未授权设备刷心跳。
3. 脚本绕过登录直接上报。
4. PC 端看到不属于自己的设备。
```

### 6.1.3 当前待修订点

当前设备绑定校验只看：

```text
user_id + device_fingerprint
```

长期建议改为：

```text
user_id + game_project_id + device_fingerprint
```

原因：

```text
1. 同一用户可能购买多个游戏。
2. 不同游戏授权设备数不同。
3. 游戏 A 的设备不应占用游戏 B 的设备名额。
```

关联待决策：

```text
T032: device_binding 是否迁移到 user_id + game_project_id + device_fingerprint 维度？
```

---

## 七、Redis Key 设计

## 7.1 心跳 Key

当前 Key：

```text
device:runtime:{game_id}:{user_id}:{device_fp}
```

示例：

```text
device:runtime:1:1001:android_device_001
```

字段含义：

```text
device:
  设备相关数据。

runtime:
  运行时状态。

game_id:
  主库 game_project.id。

user_id:
  当前用户 ID。

device_fp:
  安卓设备指纹。
```

### 7.1.1 为什么包含 game_id

```text
1. 支持多游戏隔离。
2. Celery 可以按游戏项目扫描。
3. 管理后台可以按项目查询。
4. 避免同一设备在不同游戏下混淆。
```

### 7.1.2 为什么包含 user_id

```text
1. PC 中控只拉当前用户的设备。
2. get_user_heartbeats 可按用户过滤。
3. 降低 PC 查询时后端再过滤成本。
```

### 7.1.3 为什么包含 device_fp

```text
1. 单台设备唯一定位。
2. Redis 中同一设备心跳覆盖更新。
3. Celery UPSERT 到 device_runtime.device_id。
```

---

## 7.2 心跳 Value

当前 Redis Value 是 JSON。

结构：

```json
{
  "status": "running",
  "last_seen": 1777420800,
  "game_data": {
    "current_task": "daily",
    "map": "东海岸",
    "gold": 1024
  },
  "user_id": 1001,
  "game_id": 1
}
```

字段说明：

| 字段 | 来源 | 说明 |
|---|---|---|
| `status` | AndroidScript | running / idle / error |
| `last_seen` | Verify 服务端时间 | 秒级时间戳 |
| `game_data` | AndroidScript | 游戏自定义数据 |
| `user_id` | Verify 当前用户 | 主库 user.id |
| `game_id` | Verify 当前项目 | 主库 game_project.id |

### 7.2.1 时间来源

`last_seen` 使用 Verify 服务端时间，而不是安卓端时间。

原因：

```text
1. 安卓设备时间可能不准。
2. 防止用户伪造心跳时间。
3. 统一在线判定口径。
```

---

## 7.3 TTL

当前 TTL：

```text
120 秒
```

含义：

```text
Redis Key 120 秒后自动过期。
过期后该设备不再视为实时在线。
```

与心跳间隔关系：

```text
心跳间隔: 30 秒
TTL: 120 秒
容错: 允许约 4 个心跳周期未上报
```

### 7.3.1 在线判定

```text
Redis 中存在 device:runtime key:
  is_online = true

Redis 中不存在，但游戏库 device_runtime 有记录:
  is_online = false
  source = database

Redis 和游戏库都没有:
  source = not_found
```

### 7.3.2 TTL 风险

如果网络抖动超过 120 秒：

```text
PCControl 会显示设备离线。
```

这不是数据丢失，而是在线态过期。

建议：

```text
1. PC UI 展示最后心跳时间。
2. PC UI 区分“离线”和“无历史数据”。
3. 管理后台支持按 last_seen 排序。
```

---

## 7.4 其他相关 Redis Key

当前已确认或建议统一的 Key：

```text
token:blacklist:{jti}
refresh:{user_id}:{jti}
rt_lookup:{sha256(rt_value)[:32]}
ratelimit:{endpoint_tag}:{identifier}
device:runtime:{game_id}:{user_id}:{device_fp}
```

建议后续新增独立文档：

```text
01-网络验证系统/RedisKey规范.md
```

---

## 八、PC 查询策略

## 8.1 设备列表接口

接口：

```text
GET /api/device/list
```

调用方：

```text
PCControl
```

当前数据源三层回退：

```text
第 1 层:
  Redis 心跳。
  用于在线设备。

第 2 层:
  游戏库 device_runtime。
  用于离线历史设备。

第 3 层:
  主库 DeviceBinding。
  用于已经登录绑定但还没有心跳的设备。
```

### 8.1.1 为什么需要三层回退

```text
Redis:
  实时，但会过期。

游戏库:
  历史状态，能显示最后一次运行数据。

DeviceBinding:
  登录时就创建，即使还没有心跳，也能显示设备已经绑定。
```

### 8.1.2 PC 展示建议

PCControl 设备表至少显示：

```text
1. 设备 ID。
2. 在线状态。
3. 当前状态 running / idle / error / offline。
4. 最后心跳时间。
5. 当前任务。
6. 游戏自定义摘要。
7. 来源 redis / database / binding。
```

---

## 8.2 单设备详情接口

接口：

```text
GET /api/device/data?device_fingerprint=...
```

查询顺序：

```text
1. 优先读 Redis。
2. Redis 没有则读游戏库 device_runtime。
3. 都没有则返回 source = not_found。
```

响应字段：

```json
{
  "device_id": "android_device_001",
  "user_id": 1001,
  "status": "running",
  "last_seen": "2026-04-29T00:00:00Z",
  "game_data": {},
  "is_online": true,
  "source": "redis"
}
```

### 8.2.1 source 口径

```text
redis:
  实时在线数据。

database:
  历史落库数据，设备当前可能离线。

not_found:
  Redis 和游戏库都没有记录。
```

---

## 九、Celery 批量落库策略

## 9.1 Celery 应用

文件：

```text
app/core/celery_app.py
```

Broker：

```text
Redis
```

Result Backend：

```text
Redis
```

注册任务：

```text
app.tasks.heartbeat_flush.flush_device_heartbeats
```

调度：

```text
每 30 秒执行一次
```

配置：

```python
beat_schedule = {
    "flush-heartbeats-every-30s": {
        "task": "app.tasks.heartbeat_flush.flush_device_heartbeats",
        "schedule": 30.0,
    },
}
```

### 9.1.1 Windows 开发启动

```bash
celery -A app.core.celery_app worker --pool=solo --loglevel=info
celery -A app.core.celery_app beat --loglevel=info
```

说明：

```text
Windows 不支持 fork。
所以开发环境使用 --pool=solo。
```

### 9.1.2 Linux / 生产启动

```bash
celery -A app.core.celery_app worker --concurrency=4 --loglevel=info
celery -A app.core.celery_app beat --loglevel=info
```

---

## 9.2 flush_device_heartbeats 流程

任务名：

```text
app.tasks.heartbeat_flush.flush_device_heartbeats
```

执行流程：

```text
1. Celery Beat 每 30 秒触发任务。
2. Worker 执行 flush_device_heartbeats。
3. 同步入口中调用 asyncio.run(_async_flush_all())。
4. 查询主库所有激活中的游戏项目。
5. 遍历每个游戏项目。
6. 对每个游戏项目扫描 Redis:
   device:runtime:{game_id}:*
7. 构造 device_runtime UPSERT 记录。
8. 连接该项目对应的游戏库。
9. 批量 UPSERT 到 device_runtime。
10. 单个项目失败只记录日志，不中断其他项目。
11. 返回 projects / total_devices / elapsed_ms。
```

---

## 9.3 为什么使用 NullPool

当前 Celery 任务中数据库引擎使用：

```text
NullPool
```

原因：

```text
1. flush_device_heartbeats 使用 asyncio.run()。
2. asyncio.run() 每次创建并销毁事件循环。
3. 普通连接池中的 asyncpg 连接会绑定旧事件循环。
4. 下一次任务复用旧连接时可能报 Event loop is closed。
5. NullPool 不缓存连接，每次任务新建连接，用完关闭。
```

代价：

```text
每 30 秒多一次连接建立开销。
```

当前判断：

```text
在 30 秒定时任务中可以接受。
```

---

## 9.4 为什么任务内按需创建 Redis 客户端

当前策略：

```text
Redis 客户端在 _async_flush_all() 内部创建。
任务结束后 aclose。
```

原因：

```text
避免模块级 Redis 连接池绑定到已销毁事件循环。
```

这与 NullPool 思路一致：

```text
Celery 任务不复用跨事件循环的异步连接。
```

---

## 9.5 批量 UPSERT 策略

目标表：

```text
hive_game_xxx.device_runtime
```

冲突键：

```text
device_id
```

冲突处理：

```sql
ON CONFLICT (device_id) DO UPDATE
```

更新字段：

```text
status
last_seen
game_data
updated_at
```

### 9.5.1 为什么不插入多条历史

当前 `device_runtime` 是最新状态表，不是历史流水表。

含义：

```text
同一设备只保留最新运行状态。
```

优点：

```text
1. 表规模可控。
2. PC 查询简单。
3. 后台设备监控简单。
```

限制：

```text
1. 不能回看完整心跳历史。
2. 不能分析长期运行曲线。
3. 错误状态历史会被新状态覆盖。
```

### 9.5.2 后续历史表建议

如果需要历史分析，新增：

```text
device_runtime_history
```

策略：

```text
1. 只记录状态变化。
2. 或每 5 分钟采样一次。
3. 不记录每 30 秒全量心跳。
```

---

## 十、状态字段规范

当前 `HeartbeatRequest.status` 后端 schema 限定：

```text
running
idle
error
```

建议长期状态扩展必须谨慎。

当前允许：

```text
running:
  正在运行任务。

idle:
  空闲或等待。

error:
  脚本异常。
```

不建议 AndroidScript 随意上报：

```text
starting
paused
updating
stopped
```

除非后端 schema 同步扩展。

### 10.1 当前风险

AndroidScript 文档中曾建议更多状态：

```text
starting
paused
updating
stopped
```

但 Verify 当前 schema 只允许：

```text
running
idle
error
```

### 10.2 建议方案

短期：

```text
AndroidScript 只上报 running / idle / error。
```

中期如需扩展：

```text
1. 修改 app/schemas/device.py。
2. 修改 PCControl 状态映射。
3. 修改管理后台筛选。
4. 补测试。
```

---

## 十一、性能估算

### 11.1 10,000 设备估算

```text
设备数:
  10,000 台。

心跳频率:
  30 秒一次。

Redis 写入:
  10,000 / 30 ≈ 333 次/秒。

Celery 批量:
  每 30 秒扫描约 10,000 条在线心跳。

PostgreSQL UPSERT:
  每 30 秒约 10,000 行。
  平均约 333 行/秒。
```

注：

```text
旧估算中有“约 167 行/秒”的口径，按 10,000 / 30 实际约 333 行/秒。
后续压测报告应统一用实测数据，不再依赖口算估值。
```

### 11.2 PC 查询压力估算

如果：

```text
100 个 PCControl 同时在线。
每个 10 秒拉一次设备列表。
```

则：

```text
10 次/秒设备列表查询。
```

风险：

```text
1. Redis SCAN 过多。
2. 游戏库回落查询过多。
3. 后台设备监控和 PC 查询叠加。
```

建议：

```text
1. PC 默认 10 秒同步。
2. 大客户可调到 15-30 秒。
3. 后续引入项目级在线设备缓存摘要。
4. 避免后台页面无限频率刷新。
```

---

## 十二、失败场景与处理策略

## 12.1 Redis 不可用

现象：

```text
POST /api/device/heartbeat 失败。
GET /api/device/list 无法读取在线态。
Celery 无法扫描心跳。
```

当前期望：

```text
接口返回明确错误。
不得假装心跳成功。
```

建议处理：

```text
1. API 层记录错误日志。
2. AndroidScript 进入网络异常状态。
3. PCControl 显示同步失败。
4. 管理后台显示 Redis 异常。
```

---

## 12.2 Celery Worker 未启动

现象：

```text
1. 心跳仍可写 Redis。
2. PCControl 仍可看到在线设备。
3. device_runtime 不会落库。
4. Redis TTL 过期后设备历史数据可能不更新。
```

风险：

```text
看起来在线功能正常，但历史落库断了。
```

检查：

```bash
celery -A app.core.celery_app inspect active
```

建议：

```text
生产环境必须监控 Celery Worker 存活。
```

---

## 12.3 Celery Beat 未启动

现象：

```text
Worker 存活，但定时任务不触发。
```

风险：

```text
心跳无法定时落库。
```

检查：

```text
Worker 日志中是否每 30 秒出现 flush_device_heartbeats。
```

建议：

```text
1. Worker 与 Beat 分别作为 systemd 服务。
2. 或使用可靠的 beat 调度方式。
3. 生产部署文档必须明确两个进程都要启动。
```

---

## 12.4 单个游戏库不可用

当前策略：

```text
单个项目落库失败，只记录日志，不影响其他项目继续落库。
```

优点：

```text
游戏 A 数据库异常，不影响游戏 B。
```

风险：

```text
异常项目的设备历史不更新。
```

建议：

```text
1. 对每个项目落库失败单独报警。
2. 管理后台显示项目级心跳落库状态。
3. 保留最近一次成功落库时间。
```

---

## 12.5 Redis Key 过期导致设备离线

现象：

```text
AndroidScript 停止上报超过 120 秒。
Redis Key 过期。
PCControl 看到 is_online=false。
```

这是正常行为。

建议 UI 文案：

```text
离线，最后心跳: YYYY-MM-DD HH:MM:SS
```

不要写：

```text
设备丢失
设备异常
数据不存在
```

---

## 十三、测试清单

## 13.1 单设备链路测试

```text
□ AndroidScript 登录成功。
□ device_binding 有记录。
□ AndroidScript POST /api/device/heartbeat 返回 200。
□ Redis 出现 device:runtime:{game_id}:{user_id}:{device_fp}。
□ PCControl GET /api/device/list 能看到 is_online=true。
□ 等待 30 秒。
□ Celery 日志出现 flush_device_heartbeats。
□ 游戏库 device_runtime 有记录。
□ 停止 AndroidScript 心跳。
□ 等待 120 秒以上。
□ PCControl 看到 is_online=false。
□ GET /api/device/data 返回 source=database。
```

---

## 13.2 限流测试

```text
□ 同一 user_id 1 分钟内发送 1 次心跳，返回 200。
□ 同一 user_id 1 分钟内发送 2 次心跳，返回 200。
□ 同一 user_id 1 分钟内发送 3 次心跳，返回 200。
□ 同一 user_id 1 分钟内发送 4 次心跳，返回 200。
□ 同一 user_id 1 分钟内发送第 5 次心跳，返回 429。
```

---

## 13.3 Redis TTL 测试

```text
□ 心跳写入后立即 TTL > 0。
□ TTL 初始接近 120 秒。
□ 重新心跳后 TTL 被刷新。
□ 超过 120 秒无心跳，Key 自动消失。
```

命令示例：

```bash
redis-cli -p 16379 ttl "device:runtime:1:1001:android_device_001"
```

---

## 13.4 Celery 落库测试

```text
□ Worker 启动成功。
□ Beat 启动成功。
□ Redis 中有心跳 Key。
□ 30 秒内任务触发。
□ 返回 total_devices > 0。
□ device_runtime UPSERT 成功。
□ 同一设备重复心跳不会插入重复行。
□ status 更新。
□ game_data 更新。
□ updated_at 更新。
```

---

## 13.5 多项目隔离测试

```text
□ 创建 game_001。
□ 创建 game_002。
□ 两个项目各有一台设备心跳。
□ Redis Key 中 game_id 不同。
□ Celery 分别落到 hive_game_001 和 hive_game_002。
□ game_001 的 PCControl 看不到 game_002 的设备。
□ game_002 的 PCControl 看不到 game_001 的设备。
```

---

## 13.6 游戏库异常测试

```text
□ 停止 hive_game_001 或改错连接。
□ game_001 落库失败有日志。
□ game_002 仍能落库。
□ Celery 任务整体不崩。
□ 修复 game_001 后下次任务可恢复。
```

---

## 13.7 PC 查询回退测试

```text
□ Redis 有数据时，/api/device/data 返回 source=redis。
□ Redis 过期后，游戏库有数据时，返回 source=database。
□ Redis 和游戏库都无数据时，返回 source=not_found。
□ /api/device/list 同时包含在线设备和离线设备。
□ 主库 DeviceBinding 中无心跳设备也能显示为 offline。
```

---

## 十四、压测方案

## 14.1 阶段目标

```text
阶段 1:
  100 台模拟设备。

阶段 2:
  1,000 台模拟设备。

阶段 3:
  5,000 台模拟设备。

阶段 4:
  10,000 台模拟设备。
```

### 14.1.1 第一版最低门槛

建议第一版对外测试前至少完成：

```text
1,000 台模拟设备。
```

---

## 14.2 压测指标

必须记录：

```text
1. 心跳 API 平均响应时间。
2. 心跳 API P95 响应时间。
3. 心跳 API P99 响应时间。
4. 429 数量。
5. 5xx 数量。
6. Redis CPU。
7. Redis 内存。
8. Redis Key 数。
9. Celery 每批处理设备数。
10. Celery 每批耗时。
11. PostgreSQL CPU。
12. PostgreSQL 连接数。
13. device_runtime UPSERT 耗时。
14. PCControl 设备列表接口响应时间。
```

---

## 14.3 压测通过标准

阶段 1：100 台设备

```text
□ 心跳接口无 5xx。
□ Redis Key 数符合预期。
□ Celery 每批成功落库。
□ PC 查询正常。
```

阶段 2：1,000 台设备

```text
□ P95 响应时间可接受。
□ Celery 每 30 秒批次能处理完。
□ PostgreSQL 无明显阻塞。
□ PC 查询无明显卡顿。
```

阶段 3 / 4：

```text
待实测后定义更严格标准。
```

---

## 十五、当前风险清单

## R001：心跳状态枚举不一致

已确认：

```text
后端 HeartbeatRequest.status 当前只允许:
  running / idle / error
```

风险：

```text
AndroidScript 如果上报 starting / paused / updating / stopped，会返回 422。
```

建议：

```text
短期 AndroidScript 只上报 running / idle / error。
中期如需扩展，先改后端 schema，再同步 PC 与后台。
```

---

## R002：设备绑定缺少项目维度

当前绑定校验：

```text
user_id + device_fingerprint
```

长期建议：

```text
user_id + game_project_id + device_fingerprint
```

风险：

```text
多游戏场景下设备额度可能互相影响。
```

关联：

```text
[[01-网络验证系统/风险与待决策清单]] T032
```

---

## R003：Celery 未启动时不易被发现

现象：

```text
Redis 在线态正常，PC 可以看到设备，但游戏库不落库。
```

风险：

```text
短期看起来正常，Redis 过期后历史数据缺失。
```

建议：

```text
1. 管理后台增加“最近心跳落库时间”。
2. Celery 任务失败进入日志和告警。
3. systemd 监控 Worker / Beat。
```

---

## R004：Redis SCAN 在大规模设备下需压测

当前查询使用：

```text
scan_iter("device:runtime:{game_id}:*")
scan_iter("device:runtime:{game_id}:{user_id}:*")
```

风险：

```text
设备数量很大时，频繁扫描可能有性能压力。
```

建议：

```text
1. 先压测。
2. 后续可维护项目级在线设备集合。
3. 或维护用户级在线设备集合。
```

候选 Key：

```text
device:online:set:{game_id}
device:online:user:{game_id}:{user_id}
```

当前不建议提前重构，先用压测数据判断。

---

## R005：device_runtime 只保存最新状态

当前策略：

```text
同一 device_id 只保留最新状态。
```

风险：

```text
无法回看历史心跳轨迹。
```

建议：

```text
当前保持。
后续如有分析需求，再新增 device_runtime_history。
```

---

## R006：game_data 无统一契约

当前：

```text
game_data 是 dict，各游戏自定义。
```

风险：

```text
1. PCControl 不知道如何展示。
2. 管理后台字段不稳定。
3. 不同游戏混用字段。
```

建议：

```text
每个游戏适配文档必须定义:
06-游戏适配/某游戏/设备上报字段契约.md
```

---

## 十六、建议新增决策日志

```markdown
## D023 — Verify 设备心跳采用 Redis 实时缓存 + Celery 批量落库

**日期：** 2026-04-29  
**状态：** 已确认  
**关联文档：** [[01-网络验证系统/Redis心跳落库策略]]

### 背景

蜂巢·大圣平台需要支持大量安卓设备同时运行脚本。单个游戏项目可能达到 10,000+ 设备，每台设备默认 30 秒上报一次心跳。

如果每次心跳都直接写 PostgreSQL，会给数据库造成持续高频写入压力。

### 决策

Verify 设备心跳链路采用：

```text
AndroidScript
  → POST /api/device/heartbeat
  → Redis device:runtime:{game_id}:{user_id}:{device_fp}
  → Celery 每 30 秒批量 UPSERT
  → PostgreSQL 游戏库 device_runtime
```

### 理由

```text
1. Redis 更适合高频实时状态。
2. PostgreSQL 保留稳定历史状态。
3. 心跳 API 不等待落库，响应更快。
4. PC 中控可以实时读 Redis。
5. Redis 过期后可回落游戏库历史数据。
6. 多游戏项目可独立落库，单项目失败不影响其他项目。
```

### 风险

```text
1. Celery 未启动时不会落库。
2. Redis 异常会影响在线态。
3. 大规模 SCAN 需要压测。
4. device_runtime 只保留最新状态，不保留历史心跳。
```

### 后续动作

```text
□ 完成单设备三端联调。
□ 完成 Celery Worker / Beat 运行验证。
□ 完成 1,000 台模拟设备压测。
□ 建立 Celery 任务监控。
□ 建立 Redis Key 规范文档。
```
```

---

## 十七、推荐下一步

```text
Step 1:
  把本文复制进 Obsidian:
  01-网络验证系统/Redis心跳落库策略.md

Step 2:
  运行 Redis:
  redis-cli -p 16379 ping

Step 3:
  启动 Verify:
  uvicorn app.main:app --host 0.0.0.0 --port 8000

Step 4:
  启动 Celery Worker:
  celery -A app.core.celery_app worker --pool=solo --loglevel=info

Step 5:
  启动 Celery Beat:
  celery -A app.core.celery_app beat --loglevel=info

Step 6:
  AndroidScript 登录并上报心跳。

Step 7:
  检查 Redis Key:
  device:runtime:{game_id}:{user_id}:{device_fp}

Step 8:
  PCControl 拉设备列表。

Step 9:
  等待 30 秒，检查 device_runtime。

Step 10:
  生成:
  01-网络验证系统/Redis心跳落库联调报告_2026-04-29.md
```

---

## 十八、结论

当前 Redis 心跳落库链路已经在源码层面成型。

已确认：

```text
1. 安卓心跳接口已实现。
2. 心跳限流已实现。
3. 心跳写 Redis 已实现。
4. Redis Key 和 TTL 已实现。
5. PC 列表优先读 Redis 已实现。
6. 单设备详情 Redis → 游戏库回落已实现。
7. Celery 每 30 秒批量落库已实现。
8. 游戏库 UPSERT 已实现。
9. NullPool 规避 Celery 跨事件循环连接池问题已实现。
```

当前不能直接标记为“已稳定”。

必须继续完成：

```text
1. Redis 实际运行验证。
2. Celery Worker / Beat 实际运行验证。
3. AndroidScript 心跳实机验证。
4. PCControl 设备列表联调。
5. device_runtime 落库验证。
6. 至少 1,000 台模拟设备压测。
```

当前最准确状态：

```text
Redis 心跳落库策略源码已实现；
下一步进入三端联调与压测验证。
```