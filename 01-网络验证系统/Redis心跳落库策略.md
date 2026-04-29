# Redis心跳落库策略

> **所属模块：** 01-网络验证系统
> **文档状态：** ✅ 定稿（已落地）
> **最后更新：** 2026-04-24
> **关联决策：** [[决策日志#D002]]
> **AI安全标注：** 本文所有技术参数均基于 10,000 设备规模估算，实际需根据部署环境压测调整。
>
> **变更记录：**
>
> - v1：初始版本，基于推演设计（60 秒落库、KEYS 命令、dirty:devices 脏集合）
> - v2（2026-04-24）：对齐实际代码实现 ——
>   - 落库频率 **60 秒 → 30 秒**（D2 决策）；
>   - 扫描方式 **KEYS → SCAN 游标**（已在 `redis_client.py` 中实现）；
>   - 架构变更：**移除 dirty:devices 脏集合方案**，改为 SCAN `device:runtime:{game_id}:*` 全量扫描；
>   - Celery Beat 确认为**独立进程**部署，不与 FastAPI 同进程；
>   - 关闭已落地的待办项。

---

## 一、背景与目标

### 1.1 问题描述

根据[[项目总大纲]]，安卓脚本需定时上报运行数据到云端（每 30 秒一次）。在 10,000+ 设备规模下：

- 上报请求峰值约 **333 次/秒**（假设均匀分布）
- 若每次上报都直接写入 PostgreSQL，数据库写压力巨大
- 同时需要支持 **设备在线状态判断**（超过 120 秒未上报视为离线）

### 1.2 目标

- **降低数据库写入频率**：通过 Redis 缓存批量合并写入
- **快速判断设备在线状态**：利用 Redis TTL 机制
- **保证数据最终一致性**：允许心跳数据有秒级延迟落库，但不丢失

---

## 二、整体架构

```
安卓设备 → POST /api/device/heartbeat
         │
         ▼
    FastAPI 接收请求
         │
         ├─ ① 写入 Redis（HSET + EXPIRE）
         │     Key: device:runtime:{game_id}:{user_id}:{device_fp}
         │     Fields: status, last_seen, game_data(JSON), user_id
         │
         └─ ② 立即返回 200 OK（不等待落库）


         ┌─────────────────────────────────────────┐
         │  Celery Beat 定时任务（每 30 秒，D2 决策）│
         └─────────────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────────────────────┐
         │  SCAN device:runtime:{game_id}:*        │
         │  游标扫描，避免 KEYS 阻塞 Redis          │
         └─────────────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────────────────────┐
         │  批量写入 PostgreSQL device_runtime 表   │
         │  ON CONFLICT (device_id) DO UPDATE      │
         └─────────────────────────────────────────┘
```

> **架构变更说明（v2）**：原设计使用 `dirty:devices` Set 记录脏设备，Celery 任务取集合后清空。
> 实际实现改为 **SCAN 全量扫描**，原因：
> - SCAN 天然支持游标分页，不会一次性加载所有 key
> - 每个 key 有独立 TTL（120s），过期自动清理，无需手动维护集合
> - 避免"清空集合与执行任务"之间的竞态条件

---

## 三、Redis 数据结构设计（已落地）

### 3.1 心跳数据 Hash

每个设备一个 Hash，存储最新上报的信息。

```
Key:     device:runtime:{game_id}:{user_id}:{device_fp}
Type:    Hash
Fields:
  - status      : "running" / "idle" / "error"
  - last_seen   : Unix 时间戳（秒）
  - game_data   : JSON 字符串（游戏自定义数据）
  - user_id     : 用户 ID

TTL: 120 秒（超过此时间未上报，Key 自动消失 → 设备离线）
```

**写入代码（`app/core/redis_client.py`）：**

```python
pipe = redis.pipeline()
pipe.hset(key, mapping={
    "status": status,
    "last_seen": int(time.time()),
    "game_data": json.dumps(game_data),
    "user_id": user_id,
})
pipe.expire(key, 120)
await pipe.execute()
```

> **为什么用 TTL 判断离线：**
> 设备正常每 30 秒上报一次，每次上报刷新 TTL。
> 超过 120 秒（连续 4 次上报失败）Key 自动删除，查询时 EXISTS 返回 0 = 离线。

---

## 四、批量写入 PostgreSQL 逻辑（已落地）

### 4.1 Celery 任务执行流程

实现位置：`app/tasks/heartbeat_flush.py`

```
1. 从主库查询所有 is_active=TRUE 的 game_project
2. 对每个 game_project：
   a. SCAN device:runtime:{game_id}:* 获取所有设备 key
   b. 批量 HGETALL 读取数据
   c. 过滤数据不完整的 key（防御性检查）
   d. 批量 UPSERT 到 hive_game_{code_name}.device_runtime 表
      ON CONFLICT (device_id) DO UPDATE SET status, last_seen, game_data, updated_at
3. 单个项目失败不影响其他项目
```

### 4.2 关键实现细节

**SCAN 替代 KEYS（生产安全）：**

```python
# ✅ 正确：SCAN 游标，不阻塞 Redis
async def get_all_heartbeats_for_game(redis, game_id):
    keys = []
    cursor = 0
    pattern = f"device:runtime:{game_id}:*"
    while True:
        cursor, batch = await redis.scan(cursor, match=pattern, count=100)
        keys.extend(batch)
        if cursor == 0:
            break
    ...

# ❌ 原设计：KEYS 命令，10,000 设备时会阻塞 Redis 数百毫秒
# keys = await redis.keys("device:runtime:{game_id}:*")
```

**NullPool 引擎（Celery 必须）：**

```python
# Celery 任务通过 asyncio.run() 执行，每次创建并销毁事件循环
# 必须用 NullPool，不缓存连接，避免"Event loop is closed"错误
engine = create_async_engine(url, poolclass=NullPool, connect_args={"ssl": False})
```

### 4.3 批量 Upsert 语句

```sql
INSERT INTO device_runtime (device_id, user_id, status, last_seen, game_data, updated_at)
VALUES (:device_id, :user_id, :status, :last_seen, :game_data, NOW())
ON CONFLICT (device_id) DO UPDATE SET
    status     = EXCLUDED.status,
    last_seen  = EXCLUDED.last_seen,
    game_data  = EXCLUDED.game_data,
    updated_at = NOW();
```

---

## 五、在线状态查询（PC 中控，已落地）

实现位置：`app/services/device_service.py`

```
PC 中控调用 GET /api/device/list（每 10 秒轮询，D2 决策）：
  1. 从 Redis SCAN 当前用户的所有设备 key
  2. HGETALL 批量读取在线设备数据（is_online=True）
  3. 查游戏库 device_runtime 表，找出 Redis 中不存在的设备（is_online=False）
  4. 合并返回 DeviceListResponse

响应中每台设备包含：
  device_id, user_id, status, last_seen, game_data, is_online
```

---

## 六、性能估算（10,000 设备）

|指标|计算|结果|
|---|---|---|
|心跳上报 QPS|10,000 / 30s|≈ 333 req/s|
|Redis 写操作|每次心跳 2 条命令（HSET + EXPIRE）|≈ 666 ops/s|
|Celery 落库频率|每 30 秒一次|每次处理 ≈ 10,000 条记录|
|PostgreSQL 写入 TPS|10,000 条 / 30s|≈ 333 行/s（批量 UPSERT 一次完成）|

> **结论**：单机 Redis + PostgreSQL 完全满足 Phase 1 需求。设备数增长至 50,000+ 时，考虑 Redis Cluster 和 PostgreSQL 读写分离。

---

## 七、异常与容错

|异常场景|影响|处理策略（已落地）|
|---|---|---|
|Redis 服务不可用|心跳上报失败|FastAPI 返回 503，安卓端重试|
|Celery 任务执行失败|数据延迟落库|max_retries=3，间隔 10 秒自动重试|
|单个游戏项目落库失败|该项目数据延迟|记录 ERROR 日志，继续处理其他项目|
|设备 key TTL 已过期|该设备被 SCAN 跳过|正常现象，设备已离线，跳过即可|

---

## 八、部署说明

Celery Beat 与 Worker 以**独立进程**运行，不与 FastAPI 同进程：

```bash
# FastAPI（开发环境）
uvicorn app.main:app --reload

# Celery Worker（Windows 开发环境需用 --pool=solo）
celery -A app.core.celery_app worker --pool=solo --loglevel=info

# Celery Beat（调度器，每 30 秒触发 flush_device_heartbeats）
celery -A app.core.celery_app beat --loglevel=info
```

---

## 九、待办事项

> **已关闭项（2026-04-24 对齐实现）：**
>
> - ~~评估 `KEYS` 命令在生产环境的替代方案~~ → ✅ **已落地**：改用 SCAN 游标（`redis_client.py`）
> - ~~确定 Celery Beat 调度器的部署方式~~ → ✅ **已决策**：独立进程，不与 FastAPI 同进程
> - ~~dirty:devices 脏集合方案~~ → ✅ **已废弃**：改为 SCAN 全量扫描（见 v2 架构变更说明）

**仍待推演（Phase 2+）：**

- [ ] 测试 Redis Lua 脚本原子化操作（HSET+EXPIRE 合并），减少网络往返
- [ ] 设计设备离线状态变更事件（用于主动通知 PC 中控，替代轮询）
- [ ] 压测验证 10,000 设备并发场景（Redis 和 PostgreSQL 双向压测）
