```markdown
# Redis心跳落库策略

> **所属模块：** 01-网络验证系统  
> **文档状态：** 草稿（推演定稿）  
> **最后更新：** 2026-04-16  
> **关联决策：** [[决策日志#Redis用于设备心跳缓存]]  
> **AI安全标注：** 本文所有技术参数均基于 10,000 设备规模估算，实际需根据部署环境压测调整（避免 Mirage Reasoning）。

---

## 一、背景与目标

### 1.1 问题描述

根据[[项目总大纲]]，安卓脚本需定时上报运行数据到云端（如每 30 秒一次）。在 10,000+ 设备规模下：

- 上报请求峰值约 **333 次/秒**（假设均匀分布）
- 若每次上报都直接写入 PostgreSQL，数据库写压力巨大
- 同时需要支持 **设备在线状态判断**（如超过 90 秒未上报视为离线）

### 1.2 目标

- **降低数据库写入频率**：通过 Redis 缓存批量合并写入
- **快速判断设备在线状态**：利用 Redis TTL 机制
- **保证数据最终一致性**：允许心跳数据有秒级延迟落库，但不丢失

---

## 二、整体架构

```

安卓设备 → POST /api/v1/device/heartbeat
         │
         ▼
    FastAPI 接收请求
         │
         ├─ ① 写入 Redis（Hash + TTL）
         │     Key: device:{device_id}:heartbeat
         │     Fields: status, last_seen, game_data(JSON)
         │
         └─ ② 立即返回 200 OK
               
               

         ┌─────────────────────────────────────┐
         │  Celery Beat 定时任务（每 60 秒）     │
         └─────────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────────────────┐
         │  扫描 Redis 中有更新的设备列表        │
         │  （通过额外维护的“脏标记”集合）       │
         └─────────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────────────────┐
         │  批量写入 PostgreSQL                 │
         │  device_runtime 表                  │
         │  ON CONFLICT (device_id) DO UPDATE │
         └─────────────────────────────────────┘

```
---

## 三、Redis 数据结构设计

### 3.1 心跳数据 Hash

每个设备一个 Hash，存储最新上报的信息。

```

Key:   hb:{device_id}
Type:  Hash
Fields:

  - status: "running" / "idle" / "error"
  - last_seen: 时间戳（秒）
  - data: JSON 字符串（游戏自定义数据）
  - user_id: 用户ID
  - version: 递增版本号（用于判断是否有更新）

TTL:   120 秒（超过此时间未上报，Key 自动消失 → 设备离线）

```
**示例：**
```

HSET hb:dev_001 status "running" last_seen 1713254400 data '{"gold":1000}' user_id 10086 version 5
EXPIRE hb:dev_001 120

```
> **为什么用 TTL 判断离线？**  
> 设备正常每 30 秒上报一次，每次上报会刷新 TTL（EXPIRE）。若连续两次未上报（超过 60 秒），Key 可能还存在（取决于网络波动）；若超过 120 秒仍无上报，Key 自动删除。查询在线设备时只需 `EXISTS hb:{device_id}`，极其高效。

### 3.2 脏标记集合（Set）

用于记录哪些设备在上报周期内有更新，供 Celery 批量落库。

```

Key:    dirty:devices
Type:   Set
Member: device_id

```
每次心跳上报时：
```

MULTI
HSET hb:dev_001 ...  （更新数据）
EXPIRE hb:dev_001 120
SADD dirty:devices dev_001
EXEC

```
Celery 定时任务执行时：
```

SMEMBERS dirty:devices  → 获取所有脏设备列表
DEL dirty:devices       → 清空集合（准备下一轮收集）

```
> **容错设计**：若清空集合前程序崩溃，可能导致部分设备数据未落库。但下一轮心跳会再次将设备加入脏集合，数据延迟最多 2 个周期（约 2 分钟），可接受。

---

## 四、批量写入 PostgreSQL 逻辑

### 4.1 表结构（`device_runtime`）

​```sql
CREATE TABLE device_runtime (
    device_id VARCHAR(128) PRIMARY KEY,
    user_id INTEGER NOT NULL,
    status VARCHAR(20),
    last_seen TIMESTAMP WITH TIME ZONE,
    game_data JSONB,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_device_runtime_user_id ON device_runtime(user_id);
CREATE INDEX idx_device_runtime_last_seen ON device_runtime(last_seen);
```

### 4.2 批量 Upsert 语句（PostgreSQL）

```sql
INSERT INTO device_runtime (device_id, user_id, status, last_seen, game_data, updated_at)
VALUES
    (:device_id_1, :user_id_1, :status_1, :last_seen_1, :data_1, NOW()),
    (:device_id_2, ...),
    ...
ON CONFLICT (device_id) DO UPDATE SET
    status = EXCLUDED.status,
    last_seen = EXCLUDED.last_seen,
    game_data = EXCLUDED.game_data,
    updated_at = NOW();
```

### 4.3 Celery 任务伪代码

```python
@celery_app.task
def flush_device_heartbeats():
    redis = get_redis()
    dirty_key = "dirty:devices"
    
    # 获取所有脏设备
    device_ids = redis.smembers(dirty_key)
    if not device_ids:
        return
    
    # 批量读取 Hash 数据
    pipe = redis.pipeline()
    for device_id in device_ids:
        pipe.hgetall(f"hb:{device_id}")
    results = pipe.execute()
    
    # 构造批量 upsert 数据
    records = []
    for device_id, data in zip(device_ids, results):
        if data:  # 可能 Key 已过期
            records.append({
                "device_id": device_id,
                "user_id": int(data.get("user_id", 0)),
                "status": data.get("status"),
                "last_seen": datetime.fromtimestamp(int(data.get("last_seen", 0))),
                "game_data": json.loads(data.get("data", "{}"))
            })
    
    if records:
        # 使用 SQLAlchemy 2.0 批量 upsert（需 PostgreSQL 方言支持）
        stmt = insert(DeviceRuntime).values(records)
        stmt = stmt.on_conflict_do_update(
            index_elements=['device_id'],
            set_={
                'status': stmt.excluded.status,
                'last_seen': stmt.excluded.last_seen,
                'game_data': stmt.excluded.game_data,
                'updated_at': func.now()
            }
        )
        db.execute(stmt)
        db.commit()
    
    # 清空脏集合
    redis.delete(dirty_key)
```

---

## 五、在线状态查询接口

PC 中控需要获取某用户所有设备的在线状态及最新数据。

### 5.1 方案选择

| 方案       | 说明                                        | 优缺点                                  |
| ---------- | ------------------------------------------- | --------------------------------------- |
| 纯查 Redis | 只返回当前在线的设备（Key 存在）            | 快，但离线设备信息需另查数据库          |
| 混合查询   | 先从 Redis 查在线设备，再从 DB 补全离线设备 | 实现稍复杂，但信息完整                  |
| 纯查 DB    | 直接查询 `device_runtime` 表                | 慢，且 last_seen 可能滞后（最多 60 秒） |

**采用混合查询（Phase 1 实现）：**

1. 从 Redis 通过模式匹配获取用户所有设备 Key：`KEYS hb:dev_*`（生产环境建议用 `SCAN`）
2. 批量获取在线设备数据
3. 查询数据库获取离线设备列表（`last_seen` 早于当前时间减去阈值）
4. 合并返回

> **优化方向（Phase 2）**：在 Redis 中维护一个 `user:{user_id}:devices` Set，记录用户拥有的所有设备 ID，避免 SCAN 全局。

---

## 六、异常与容错

| 异常场景                 | 影响         | 处理策略                                                     |
| ------------------------ | ------------ | ------------------------------------------------------------ |
| Redis 服务不可用         | 心跳上报失败 | 返回 503，安卓端重试（指数退避）                             |
| Celery 任务执行失败      | 数据延迟落库 | Celery 自动重试机制；监控告警                                |
| 批量写入时部分记录失败   | 部分数据丢失 | 事务回滚？**不采用**。改用逐条插入 + 错误日志，保证其余数据落库 |
| 脏集合过大（设备数暴涨） | 内存占用高   | 设置最大成员数告警；分批处理                                 |

---

## 七、性能估算（10,000 设备）

| 指标                | 计算                                  | 结果                             |
| ------------------- | ------------------------------------- | -------------------------------- |
| 心跳上报 QPS        | 10,000 / 30s ≈                        | 333 req/s                        |
| Redis 写操作        | 每次心跳 3 条命令（HSET+EXPIRE+SADD） | ~1000 ops/s                      |
| 脏集合大小（平均）  | 每 60 秒内上报过的设备数              | 接近全部设备（因上报间隔 30s）   |
| 批量写入频率        | 每 60 秒一次                          | 每次处理 ~10,000 条记录          |
| PostgreSQL 写入 TPS | 10,000 条 / 60s ≈                     | 167 行/s（批量 upsert 一次完成） |

> **结论**：在单机 Redis + PostgreSQL 场景下，完全满足 Phase 1 需求。若设备数增长至 50,000+，可考虑 Redis Cluster 和 PostgreSQL 读写分离。

---

## 八、与 API 鉴权的配合

心跳接口需携带 JWT Access Token，通过 `get_current_user` 依赖项获取 `user_id`，确保上报数据归属于正确用户。

```python
@router.post("/device/heartbeat")
async def device_heartbeat(
    request: HeartbeatRequest,
    current_user: User = Depends(get_current_user),
    redis: Redis = Depends(get_redis)
):
    device_id = request.device_id
    # 权限校验：该设备是否绑定给 current_user
    # ...
    # 写入 Redis
    pipe = redis.pipeline()
    pipe.hset(f"hb:{device_id}", mapping={
        "status": request.status,
        "last_seen": int(time.time()),
        "data": json.dumps(request.game_data),
        "user_id": current_user.id,
        "version": int(time.time() * 1000)  # 毫秒级版本
    })
    pipe.expire(f"hb:{device_id}", 120)
    pipe.sadd("dirty:devices", device_id)
    await pipe.execute()
    
    return {"code": 0, "msg": "ok"}
```

---

## 九、待办事项

- [ ] 测试 Redis Lua 脚本原子化操作（HSET+EXPIRE+SADD），减少网络往返
- [ ] 设计设备离线状态变更事件（用于通知 PC 中控）
- [ ] 评估 `KEYS` 命令在生产环境的替代方案（SCAN 游标）
- [ ] 确定 Celery Beat 调度器的部署方式（是否与 FastAPI 同进程）
- [ ] 压测验证 10,000 设备并发场景