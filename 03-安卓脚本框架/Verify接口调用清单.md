---
文件位置: 03-安卓脚本框架/Verify接口调用清单.md
名称: AndroidScript Verify 接口调用清单
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-05-12
版本: V1.0.1
状态: 草稿
关联文档:
  - "[[项目总大纲]]"
  - "[[01-网络验证系统/架构设计]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/三端联调测试清单]]"
  - "[[01-网络验证系统/风险与待决策清单]]"
  - "[[03-安卓脚本框架/架构设计]]"
  - "[[03-安卓脚本框架/懒人精灵接口规范]]"
  - "[[03-安卓脚本框架/热更新方案]]"
  - "[[编码规范]]"
  - "[[编码规范_补充_Markdown文档规范]]"
变更记录:
  - V1.0.0: 基于 HiveGreatSage-AndroidScript 当前源码反向整理安卓端调用 Verify 的接口、启动流程、字段契约、风险点与联调清单
---

# AndroidScript Verify 接口调用清单

← 返回 [[00-项目总控/项目总大纲]] | 父节点: [[03-安卓脚本框架/架构设计]]

## 当前证据边界

- 本文是 AndroidScript 调用 Verify 接口关系的文档清单。
- 本文包含历史源码反向整理结论，但本轮未重新逐行复核 AndroidScript 当前 main 源码。
- 本文不代表所有接口已完成实机登录、心跳落库、参数同步、热更新安装或三端联调。
- "已确认 / 当前源码已经实现"仅表示当时治理批次登记状态。
- 涉及发布、联调、闭环判断时，必须重新对照 AndroidScript main 源码、Verify main 源码、测试结果、懒人精灵真机运行结果和 PCControl 设备展示结果。

## 一、文档目的

本文用于沉淀 `HiveGreatSage-AndroidScript` 当前安卓端脚本与 `HiveGreatSage-Verify` 网络验证系统之间的接口调用关系。

本文不是 Verify 后端接口总表，而是从 **安卓脚本调用方视角** 反向整理：

```text
1. AndroidScript 当前调用了哪些 Verify 接口。
2. 每个接口由哪个 Lua 模块调用。
3. 每个接口依赖哪些配置项。
4. 每个接口的请求字段是什么。
5. 当前源码已经实现什么。
6. 当前源码还缺什么。
7. 哪些问题会影响三端联调。
8. 后续应该如何测试和修订。
```

---

## 二、当前登记结论

### 2.1 历史登记为已接入的链路

当前 AndroidScript 已经接入 Verify 的核心链路：

```text
登录验证:
  framework/verify.lua
  POST /api/auth/login

Token 刷新:
  framework/verify.lua
  POST /api/auth/refresh

登出:
  framework/verify.lua
  POST /api/auth/logout

设备心跳:
  framework/heartbeat.lua
  POST /api/device/heartbeat

参数拉取:
  framework/cloud_sync.lua
  GET /api/params/get

参数保存:
  framework/cloud_sync.lua
  POST /api/params/set

热更新检查:
  framework/updater.lua
  GET /api/update/check?client_type=android&current_version=...

热更新下载:
  framework/updater.lua
  GET /api/update/download?client_type=android
```

当前主入口启动顺序中已经包含：

```text
1. UI 输入账号、密码、服务器地址、PC 中控 IP。
2. 设置 Config.API_BASE_URL。
3. 分辨率适配。
4. 代理/VPN。
5. OCR 初始化。
6. 登录 Verify。
7. 热更新检查。
8. 释放/提取资源。
9. 拉取云端参数。
10. 启动心跳。
11. 连接 PC 中控。
12. 看门狗。
13. 随机延时。
14. 调度器。
15. 游戏主逻辑。
```

### 2.2 待确认

以下内容当前不能写成已完成：

```text
1. 安卓端是否已经实机成功登录 Verify。
2. 安卓端心跳是否已被 Verify 写入 Redis。
3. Celery 是否已将心跳批量落库到 device_runtime。
4. PCControl 是否能看到安卓端上报的设备。
5. AndroidScript 是否能正确解析 /api/params/get 返回参数。
6. AndroidScript 是否能真实下载 lrj 更新包。
7. installLrPkg 是否在目标设备和运行模式下成功。
8. 当前热更新是否校验 SHA-256。
9. 当前强制更新失败是否会阻断继续运行。
10. IMSI 上传接口是否需要由 AndroidScript 主动调用。
```

### 2.3 当前风险最高的点

```text
1. OCR_LICENSE 当前写在 config.lua 中，存在敏感配置入库风险。
2. 热更新下载后先写入本地版本号，再 installLrPkg，失败时可能造成本地版本状态不准确。
3. 当前 Updater 未见 SHA-256 校验。
4. Heartbeat 遇到 401 会刷新 Token，但本次心跳不会立即重试。
5. CloudSync / Updater 遇到 401 当前没有统一 ensure_token 重试逻辑。
6. Verify.get_fingerprint 会尝试读取 IMSI，但当前未见调用 /api/device/imsi 主动上传 IMSI。
7. Config.API_BASE_URL 依赖 UI 或 KV 输入，三端联调前必须固定测试地址。
```

---

## 三、相关源码文件

### 3.1 配置文件

```text
脚本/config.lua
```

职责：

```text
1. 保存游戏项目基础配置。
2. 保存 Verify API 默认地址。
3. 保存 PROJECT_UUID。
4. 保存 OCR / YOLO 授权配置。
5. 保存心跳间隔。
6. 保存脚本版本号。
```

当前 Verify 相关配置：

```lua
Config.API_BASE_URL  = ""
Config.PROJECT_UUID = "<project_uuid_example_or_test_value>"
Config.HEARTBEAT_INTERVAL = 30
Config.SCRIPT_VERSION = "1.0.0"
Config.LOG_TO_CLOUD = true
```

### 3.2 登录验证模块

```text
脚本/framework/verify.lua
```

职责：

```text
1. 获取设备指纹。
2. 登录 Verify。
3. 保存 Access Token / Refresh Token。
4. 刷新 Token。
5. 自动重新登录。
6. 登出 Verify。
```

### 3.3 心跳模块

```text
脚本/framework/heartbeat.lua
```

职责：

```text
1. 维护当前 status。
2. 维护当前 game_data。
3. 独立线程定时上报心跳。
4. 401 时尝试刷新 Token。
```

### 3.4 云端参数同步模块

```text
脚本/framework/cloud_sync.lua
```

职责：

```text
1. 拉取脚本参数。
2. 保存脚本参数。
3. 统一携带 Authorization Header。
```

### 3.5 热更新模块

```text
脚本/framework/updater.lua
```

职责：

```text
1. 读取当前本地脚本版本。
2. 调用 Verify 检查更新。
3. 获取下载链接。
4. 下载 lrj。
5. 调用 installLrPkg。
6. 重启脚本。
```

### 3.6 主入口

```text
脚本/HiveGreatSage-AndroidScript.lua
```

职责：

```text
1. 加载插件。
2. 注册停止回调。
3. 显示启动 UI。
4. 设置 API 地址。
5. 登录 Verify。
6. 热更新。
7. 拉参数。
8. 启动心跳。
9. 连接 PC 中控。
10. 启动游戏主逻辑。
```

---

## 四、Verify 配置契约

## 4.1 `Config.API_BASE_URL`

### 当前用途

所有 Verify 请求都拼接：

```lua
Config.API_BASE_URL .. path
```

示例：

```lua
Config.API_BASE_URL .. "/api/auth/login"
Config.API_BASE_URL .. "/api/device/heartbeat"
Config.API_BASE_URL .. "/api/params/get"
Config.API_BASE_URL .. "/api/update/check?client_type=android&current_version=1.0.0"
```

### 当前来源

主入口中：

```text
优先从 UI 输入 edit_api_url 读取；
如果为空，读取 KV 中 hive_api_url；
如果仍为空，读取 Config.API_BASE_URL；
如果仍为空，提示“请先填写服务器地址”并退出。
```

### 建议规则

```text
开发调试:
  http://局域网IP:8000

生产环境:
  https://verify.example.com

禁止:
  在业务模块中写死 API 地址。
```

### 风险点

```text
1. API_BASE_URL 末尾是否带 / 需要统一。
2. 当前拼接方式要求 API_BASE_URL 不带尾部 /。
3. 如果 UI 输入 http://x.x.x.x:8000/，可能出现双斜杠。
```

### 建议方案

新增统一清洗函数：

```lua
local function normalize_base_url(url)
    url = tostring(url or ""):match("^%s*(.-)%s*$")
    url = url:gsub("/+$", "")
    return url
end
```

---

## 4.2 `Config.PROJECT_UUID`

### 当前用途

登录 Verify 时作为项目身份：

```lua
project_uuid = Config.PROJECT_UUID
```

### 当前值

```lua
Config.PROJECT_UUID = "07238db5-129a-4408-b82a-e025be4652a1"
```

### 设计要求

```text
1. PROJECT_UUID 必须与 Verify 中 game_project.project_uuid 一致。
2. PCControl 与 AndroidScript 必须使用同一项目 UUID。
3. 不同游戏 fork 后必须修改 PROJECT_UUID。
4. PROJECT_UUID 不应由用户随意输入。
```

### 风险点

```text
1. fork 新游戏时忘记修改 PROJECT_UUID，会登录到错误项目。
2. PCControl 与 AndroidScript 项目 UUID 不一致，会导致设备数据无法互通。
```

### 建议方案

```text
1. 新游戏 fork checklist 中加入 PROJECT_UUID 必改项。
2. 在启动日志中打印 project_uuid 的前 8 位。
3. 管理后台导出项目配置时包含 project_uuid。
```

---

## 4.3 `Config.HEARTBEAT_INTERVAL`

### 当前用途

心跳线程间隔：

```lua
Config.HEARTBEAT_INTERVAL = 30
```

### 与 Verify 的关系

Verify 端当前心跳限流口径：

```text
同 user_id 每分钟最多 4 次。
```

30 秒一次等于每分钟 2 次，符合限流。

### 风险点

```text
如果用户或后续开发者把心跳间隔改为小于 15 秒，可能触发 429。
```

### 建议方案

```text
1. 在 Heartbeat.start() 中限制最小间隔。
2. 推荐最小值 20 秒。
3. 默认值保持 30 秒。
```

---

## 4.4 `Config.SCRIPT_VERSION`

### 当前用途

热更新检查时作为当前版本：

```lua
Config.SCRIPT_VERSION = "1.0.0"
```

Updater 第一次运行时写入：

```lua
KEY_VERSION = "hive_script_version"
```

### 风险点

```text
1. 当前本地版本可能来自 KV，而不是 Config.SCRIPT_VERSION。
2. 如果 installLrPkg 失败但已写入新版本，会造成版本误判。
```

### 建议方案

```text
1. 下载成功后不立即写 KEY_VERSION。
2. installLrPkg 成功或新包启动成功后再写入版本。
3. 或由新包启动时校验并写入自己的 Config.SCRIPT_VERSION。
```

---

## 五、设备指纹契约

## 5.1 当前指纹优先级

`Verify.get_fingerprint()` 当前优先级：

```text
1. 老狼孩插件:
   设备.取硬件序列号()

2. 懒人精灵内置:
   getSubscriberId()

3. WiFi MAC:
   getWifiMac()

4. 兜底:
   "fb_" .. os.time()
```

### 5.2 登录请求字段

```lua
device_fingerprint = fp
```

### 5.3 心跳请求字段

```lua
device_fingerprint = Verify.get_fingerprint()
```

### 5.4 风险点

```text
1. 时间戳兜底不稳定，重启后可能变化。
2. IMSI 属于敏感信息，是否作为指纹需要合规确认。
3. WiFi MAC 在部分安卓版本可能不可读或固定返回。
4. 硬件序列号依赖插件和运行模式。
```

### 5.5 建议方案

```text
1. 首次获取稳定指纹后写入 KV。
2. 后续优先读取 KV 中 hive_device_fingerprint。
3. 只有 KV 为空时才重新探测。
4. 时间戳兜底生成后也必须持久化。
5. UI 或日志中只显示指纹后 6 位。
```

### 5.6 推荐改造口径

```lua
local KEY_DEVICE_FINGERPRINT = "hive_device_fingerprint"

function Verify.get_fingerprint()
    if _fingerprint then return _fingerprint end

    local saved = readKeyVal(KEY_DEVICE_FINGERPRINT)
    if saved and saved ~= "" then
        _fingerprint = saved
        return _fingerprint
    end

    -- 再执行插件 / IMSI / MAC / 时间戳探测
    -- 探测成功后 writeKeyVal(KEY_DEVICE_FINGERPRINT, _fingerprint)
end
```

---

## 六、认证接口调用

## 6.1 登录接口

### Lua 模块

```text
脚本/framework/verify.lua
```

### Lua 函数

```lua
Verify.login(username, password)
```

### Verify 接口

```text
POST /api/auth/login
```

### 请求体

```json
{
  "username": "<username_from_input>",
  "password": "<password_from_secure_input>",
  "project_uuid": "<project_uuid_example_or_test_value>",
  "device_fingerprint": "<device_fingerprint_example>",
  "client_type": "android"
}
```

### 当前源码请求字段

```lua
{
    username           = username,
    password           = password,
    project_uuid       = Config.PROJECT_UUID,
    device_fingerprint = fp,
    client_type        = "android",
}
```

### 成功处理

当前源码成功后：

```text
1. 保存 access_token。
2. 保存 refresh_token。
3. 写入 hive_access_token。
4. 写入 hive_refresh_token。
5. 返回 true。
```

### 失败处理

当前源码失败时：

```text
1. 无响应 → 返回 false, "网络请求失败"。
2. JSON 解析失败 → 返回 false, "响应解析失败"。
3. 无 access_token → 返回 false, detail。
```

### 主入口处理

主入口中：

```text
登录失败:
  toast("登录失败: " .. login_err)
  exitScript()

登录成功:
  writeKeyVal("last_login_time", os.time())
```

### 风险点

```text
1. 登录失败 detail 可能是 table，当前 tostring(detail) 可能不够友好。
2. 未区分 401 / 403 / 404 / 429。
3. 未对登录限流 429 做专门提示。
4. 未对服务器地址错误做明确提示。
```

### 建议方案

统一错误码映射：

```text
401:
  账号或密码错误。

403:
  授权无效、账号停用、设备超限。

404:
  项目不存在或服务器地址错误。

429:
  登录过于频繁，请稍后再试。

timeout:
  网络连接失败，请检查服务器地址。
```

---

## 6.2 Refresh Token 接口

### Lua 模块

```text
脚本/framework/verify.lua
```

### Lua 函数

```lua
Verify.refresh_token()
```

### Verify 接口

```text
POST /api/auth/refresh
```

### 请求体

```json
{
  "refresh_token": "..."
}
```

### 当前行为

```text
1. 从 KV 读取 hive_refresh_token。
2. 调用 /api/auth/refresh。
3. 成功后更新 access_token。
4. 写入 hive_access_token。
```

### 风险点

```text
1. 只更新 access_token，没有看到 refresh_token 轮换保存。
2. 如果 Verify 后端采用 refresh token rotation，需要同步保存新的 refresh_token。
3. 调用方只有 Heartbeat 在 401 时触发 ensure_token。
4. CloudSync 和 Updater 当前没有统一 401 刷新重试。
```

### 建议方案

`refresh_token()` 兼容后端返回新 refresh_token：

```lua
if data.refresh_token then
    writeKeyVal(KEY_REFRESH_TOKEN, data.refresh_token)
    _refresh_token = data.refresh_token
end
```

---

## 6.3 自动确保 Token

### Lua 函数

```lua
Verify.ensure_token()
```

### 当前行为

```text
1. 优先尝试 refresh_token。
2. 如果 refresh 失败，从 KV 读取 username/password。
3. 密码如果是加密格式，则用设备指纹解密。
4. 调用 Verify.login 重新登录。
```

### 当前优点

```text
1. 支持 Token 过期后自动恢复。
2. 密码不是直接明文存储。
3. 兼容旧版明文密码。
```

### 风险点

```text
1. 密码仍然以可逆加密形式保存在本地 KV。
2. 设备指纹变化会导致密码无法解密。
3. 如果 Verify.get_fingerprint 兜底时间戳变化，解密会失败。
```

### 建议方案

```text
1. 设备指纹必须持久化。
2. 密码本地保存应尽量避免，长期改为 Refresh Token 续期。
3. 如果解密失败，应提示用户重新登录，而不是静默失败。
```

---

## 6.4 登出接口

### Lua 函数

```lua
Verify.logout()
```

### Verify 接口

```text
POST /api/auth/logout
```

### 当前请求体

```lua
{
    refresh_token = _refresh_token or ""
}
```

### 当前行为

```text
1. 调用 /api/auth/logout。
2. 清空 access_token。
3. 清空 refresh_token。
4. 写日志。
```

### 风险点

```text
1. 如果 _refresh_token 没有先 _load_tokens，可能传空。
2. 登出失败也会清空本地 token。
3. 当前主入口未见退出时主动 logout，只在 stop callback 做清理。
```

### 建议方案

```text
1. logout 前先 _load_tokens。
2. 用户主动退出时调用 logout。
3. 脚本被系统杀死时不能依赖 logout。
```

---

## 6.5 当前未调用的认证接口

### `/api/auth/me`

当前状态：

```text
AndroidScript 当前未见调用。
```

建议用途：

```text
1. 登录后验证 Token 是否有效。
2. 显示当前用户等级、授权到期、项目名。
3. 用于调试登录状态。
```

是否必须：

```text
非必须。
```

### `/api/auth/revoke-all`

当前状态：

```text
AndroidScript 当前未见调用。
```

建议用途：

```text
不建议由 AndroidScript 主动调用。
应由 PCControl 或用户后台触发。
```

---

## 七、心跳接口调用

## 7.1 模块与函数

### Lua 模块

```text
脚本/framework/heartbeat.lua
```

### 函数

```lua
Heartbeat.start()
Heartbeat.stop()
Heartbeat.set_status(s)
Heartbeat.set_game_data(d)
```

### Verify 接口

```text
POST /api/device/heartbeat
```

---

## 7.2 请求体

当前源码请求体：

```lua
{
    device_fingerprint = Verify.get_fingerprint(),
    status             = _status,
    game_data          = _game_data,
}
```

JSON 示例：

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

### Header

```text
Content-Type: application/json
Authorization: Bearer {access_token}
```

---

## 7.3 当前线程模型

当前 Heartbeat 使用：

```lua
beginThread(function()
    while _running do
        _beat()
        sleep(Config.HEARTBEAT_INTERVAL * 1000)
    end
end)
```

这是正确方向。

原因：

```text
旧 setTimer 方案会被主线程 while true do sleep() end 阻塞。
beginThread 独立线程可以避免主线程阻塞导致心跳不触发。
```

---

## 7.4 当前响应处理

当前处理逻辑：

```text
code == 401:
  Token 过期，调用 Verify.ensure_token()

code == 200:
  记录上报成功

其他:
  记录上报失败 code
```

### 风险点

```text
1. 401 刷新成功后，本次心跳不会立即重发。
2. 403 不区分设备未绑定、授权失效、设备超限。
3. 429 不区分心跳限流。
4. 5xx 不做退避。
5. 没有连续失败计数。
```

### 建议方案

增加连续失败计数：

```text
1. 连续失败 3 次，toast 提示网络异常。
2. 连续 401 且 ensure_token 失败，进入重新登录状态。
3. 429 时拉长心跳间隔。
4. 5xx 时指数退避。
```

---

## 7.5 心跳状态口径

建议统一 `_status` 枚举：

```text
idle       空闲
starting   启动中
running    运行中
paused     暂停
error      错误
updating   更新中
stopped    停止
```

当前风险：

```text
_status 是任意字符串。
```

建议：

```lua
local VALID_STATUS = {
    idle = true,
    starting = true,
    running = true,
    paused = true,
    error = true,
    updating = true,
    stopped = true,
}
```

---

## 7.6 game_data 推荐结构

建议 AndroidScript 上报：

```json
{
  "task": {
    "name": "daily",
    "stage": "fight",
    "progress": 35
  },
  "game": {
    "map": "东海岸",
    "role_level": 35,
    "gold": 1024
  },
  "runtime": {
    "script_version": "1.0.0",
    "uptime_sec": 3600,
    "last_error": ""
  }
}
```

短期最小字段：

```text
current_task
map
gold
level
last_error
```

### 风险点

```text
game_data 是 JSON 自由字段。
如果没有约束，PCControl 表格和后台设备监控会难以稳定展示。
```

### 建议方案

每个游戏适配文档中必须定义：

```text
06-游戏适配/某游戏/设备上报字段契约.md
```

---

## 八、参数同步接口调用

## 8.1 拉取参数

### Lua 模块

```text
脚本/framework/cloud_sync.lua
```

### 函数

```lua
CloudSync.pull_params()
```

### Verify 接口

```text
GET /api/params/get
```

### Header

```text
Authorization: Bearer {access_token}
```

### 当前返回处理

当前源码：

```lua
local params = data.params or data or {}
return params
```

### 兼容性

当前兼容两种响应：

```json
{
  "params": {
    "auto_task": true
  }
}
```

以及：

```json
{
  "auto_task": true
}
```

### 风险点

```text
1. 响应结构过于宽松，可能掩盖后端契约漂移。
2. 参数类型可能是字符串，也可能是 bool/int。
3. 未处理 401 自动刷新。
4. 未处理 403 授权失效。
5. 未处理 422 响应。
```

### 建议方案

统一后端响应结构：

```json
{
  "params": {
    "auto_task": "true",
    "target_map": "东海岸",
    "max_gold": "2000"
  },
  "version": 1,
  "updated_at": "2026-04-29T00:00:00"
}
```

AndroidScript 固定解析：

```lua
local params = data.params or {}
```

不再兼容裸对象。

---

## 8.2 保存参数

### Lua 函数

```lua
CloudSync.save_params(params_table)
```

### Verify 接口

```text
POST /api/params/set
```

### 请求体

```json
{
  "params": {
    "auto_task": "true",
    "target_map": "东海岸"
  }
}
```

### 当前用途

已确认源码存在该函数。

待确认：

```text
AndroidScript 主流程当前是否实际调用 save_params。
```

长期建议：

```text
参数修改主要由 PCControl 发起；
AndroidScript 一般只拉取参数，不保存参数。
```

### 风险点

```text
如果 AndroidScript 也能保存参数，需明确：
1. 是否允许安卓端覆盖 PC 设置。
2. 是否需要权限控制。
3. 是否需要参数变更来源字段。
```

### 建议方案

```text
1. AndroidScript 保留 save_params 作为调试能力。
2. 生产逻辑默认不调用。
3. PCControl 作为参数修改主入口。
```

---

## 九、热更新接口调用

## 9.1 检查更新

### Lua 模块

```text
脚本/framework/updater.lua
```

### 函数

```lua
Updater.check_and_update()
```

### Verify 接口

```text
GET /api/update/check?client_type=android&current_version={current}
```

### 当前版本来源

```text
优先读取 KV:
  hive_script_version

如果没有:
  写入 Config.SCRIPT_VERSION
```

### 当前请求示例

```text
GET /api/update/check?client_type=android&current_version=1.0.0
```

### 当前响应处理

```text
need_update = false:
  记录已是最新版本，返回。

need_update = true:
  记录发现新版本。
  如果 update_message 不为空，toast。
  调用 /api/update/download。
```

### 风险点

```text
1. 当前使用 check.latest_version。
2. 需要确认 Verify 实际返回字段是否是 latest_version 还是 version。
3. update_message 字段需确认后端是否返回。
4. force_update 字段只在下载链接失败时处理，下载失败时未强制退出。
```

### 建议方案

Verify 响应字段统一为：

```json
{
  "need_update": true,
  "latest_version": "1.0.1",
  "force_update": false,
  "update_message": "更新说明",
  "checksum_sha256": "..."
}
```

或者统一为：

```json
{
  "need_update": true,
  "version": "1.0.1",
  "force_update": false,
  "release_notes": "更新说明",
  "checksum_sha256": "..."
}
```

二选一，不要混用。

---

## 9.2 下载更新

### Verify 接口

```text
GET /api/update/download?client_type=android
```

### 当前响应要求

当前源码期望：

```json
{
  "download_url": "https://..."
}
```

### 当前下载路径

```lua
local save_path = getWorkPath() .. "update.lrj"
```

### 当前安装方式

```lua
installLrPkg(save_path)
restartScript()
```

注意：

```text
函数名是 installLrPkg，小写 r。
不是 installLRPkg。
```

---

## 9.3 热更新当前高风险点

### R001 未见 SHA-256 校验

当前源码下载后直接安装：

```lua
local dl_ret = downloadFile(dl.download_url, save_path)
installLrPkg(save_path)
```

风险：

```text
1. 文件下载损坏仍可能安装。
2. 中间替换无法被客户端发现。
3. Verify 返回 checksum_sha256 没有被客户端使用。
```

建议方案：

```text
1. Verify /api/update/check 或 /api/update/download 返回 checksum_sha256。
2. AndroidScript 下载后计算 SHA-256。
3. 不一致则拒绝安装。
```

待确认：

```text
懒人精灵环境中是否有可用 SHA-256 函数。
如果没有，需要通过插件或后端签名方案处理。
```

---

### R002 本地版本写入时机过早

当前源码：

```lua
writeKeyVal(KEY_VERSION, check.latest_version or current)
installLrPkg(save_path)
restartScript()
```

风险：

```text
如果 installLrPkg 失败，本地 KV 已经写成新版本。
下次启动可能误以为已更新。
```

建议方案：

```text
1. 下载成功后不写版本。
2. 安装成功或新脚本启动后写版本。
3. 新脚本启动时用 Config.SCRIPT_VERSION 覆盖 KEY_VERSION。
```

---

### R003 强制更新失败处理不完整

当前源码：

```text
获取下载链接失败且 force_update=true:
  toast 后 exitScript。

downloadFile 失败:
  只 return，没有根据 force_update exitScript。
```

建议方案：

```text
如果 force_update=true，以下失败都应阻断继续运行：
1. 获取下载链接失败。
2. 下载失败。
3. 校验失败。
4. 安装失败。
```

---

### R004 下载路径覆盖

当前固定保存：

```text
getWorkPath() .. "update.lrj"
```

风险：

```text
1. 多次下载覆盖。
2. 失败包残留。
3. 无版本号不利于排查。
```

建议方案：

```lua
local save_path = getWorkPath() .. "update_" .. tostring(check.latest_version) .. ".lrj"
```

---

## 十、当前未调用的 Verify 接口

## 10.1 `/api/device/imsi`

### 后端已有接口

Verify 已有：

```text
POST /api/device/imsi
```

### AndroidScript 当前状态

当前源码未见主动调用该接口。

虽然 `Verify.get_fingerprint()` 会尝试：

```lua
getSubscriberId()
```

但这是作为设备指纹来源之一，不等于调用 `/api/device/imsi` 上传。

### 待决策

```text
T-AS-001:
AndroidScript 是否需要主动上传 IMSI？
```

### 建议方案

默认不主动上传 IMSI。

如果确实需要：

```text
1. 增加 Config.ENABLE_IMSI_UPLOAD = false。
2. 只有开启时才调用 /api/device/imsi。
3. 上传前脱敏或 hash。
4. 后台默认脱敏展示。
```

---

## 10.2 `/api/auth/me`

当前未见调用。

建议用途：

```text
1. 登录后调试。
2. Token 恢复后确认账号状态。
3. 显示用户授权信息。
```

是否必须：

```text
当前不是必须。
```

---

## 10.3 `/api/auth/revoke-all`

当前未见调用。

建议：

```text
不由 AndroidScript 调用。
应由 PCControl 或管理后台触发。
```

---

## 十一、主入口 Verify 链路

## 11.1 当前启动流程

主入口中 Verify 相关流程：

```text
Step 1:
  读取 UI 输入。

Step 2:
  设置 Config.API_BASE_URL。

Step 3:
  初始化分辨率。

Step 4:
  启动代理/VPN。

Step 5:
  初始化 OCR。

Step 6:
  Verify.login(username, password)。

Step 7:
  Updater.check_and_update()。

Step 8:
  Finder.extract_assets("HiveGreatSage-AndroidScript.rc")。

Step 9:
  CloudSync.pull_params()。

Step 10:
  Heartbeat.start()。

Step 11:
  CommLan.connect(lan_ip)。

Step 12:
  ErrorHandler.init(Config.GAME_PACKAGE)。

Step 13:
  Scheduler.run()。

Step 14:
  game_main.run(params)。
```

## 11.2 当前顺序评价

### 已确认合理点

```text
1. 先登录 Verify，再热更新。
2. 先热更新，再进入游戏逻辑。
3. 先拉参数，再运行 game_main。
4. 心跳在登录成功后启动。
5. 停止回调中会停止 Heartbeat。
```

### 待确认点

```text
1. 热更新安装后是否会立即 restartScript，后续流程是否还会继续执行。
2. Finder.extract_assets 是否应放在热更新之后，当前顺序是合理的。
3. CloudSync.pull_params 失败时是否应该阻断脚本运行，当前返回空表继续运行。
4. Heartbeat.start 后是否应立即设置 status="running"。
5. game_main.run 异常后，心跳是否应设置 status="error"。
```

### 建议方案

在主入口中明确状态：

```lua
Heartbeat.set_status("starting")

-- 登录成功
Heartbeat.set_status("updating")
Updater.check_and_update()

Heartbeat.set_status("running")
Heartbeat.start()

local ok, err = pcall(function()
    game_main.run(params)
end)

if not ok then
    Heartbeat.set_status("error")
    Heartbeat.set_game_data({ last_error = tostring(err) })
end
```

---

## 十二、字段契约总表

## 12.1 AndroidScript → Verify 登录字段

| 字段 | 当前来源 | 必填 | 说明 |
|---|---|---|---|
| `username` | UI 输入 | 是 | 用户账号 |
| `password` | UI 输入 | 是 | 用户密码 |
| `project_uuid` | `Config.PROJECT_UUID` | 是 | 游戏项目 UUID |
| `device_fingerprint` | `Verify.get_fingerprint()` | 是 | 安卓设备指纹 |
| `client_type` | 固定 `"android"` | 是 | 安卓端登录 |

## 12.2 AndroidScript → Verify 心跳字段

| 字段 | 当前来源 | 必填 | 说明 |
|---|---|---|---|
| `device_fingerprint` | `Verify.get_fingerprint()` | 是 | 当前设备指纹 |
| `status` | `Heartbeat.set_status()` | 是 | 当前运行状态 |
| `game_data` | `Heartbeat.set_game_data()` | 否 | 游戏运行数据 |

## 12.3 AndroidScript → Verify 参数拉取

| 字段 | 当前来源 | 必填 | 说明 |
|---|---|---|---|
| `Authorization` | `Verify.get_token()` | 是 | Bearer Token |

## 12.4 AndroidScript → Verify 热更新检查

| 字段 | 当前来源 | 必填 | 说明 |
|---|---|---|---|
| `client_type` | 固定 `android` | 是 | 客户端类型 |
| `current_version` | KV 或 `Config.SCRIPT_VERSION` | 是 | 当前脚本版本 |

## 12.5 Verify → AndroidScript 热更新响应

当前源码使用字段：

| 字段 | 当前用途 | 必填 | 风险 |
|---|---|---|---|
| `need_update` | 判断是否更新 | 是 | 必须稳定 |
| `latest_version` | 新版本号 | 是 | 需确认后端字段 |
| `update_message` | toast 提示 | 否 | 需确认后端字段 |
| `force_update` | 强制更新 | 否 | 当前处理不完整 |
| `download_url` | 下载链接 | 是 | 在 download 接口返回 |

建议补充字段：

```text
checksum_sha256
package_size
expires_at
```

---

## 十三、错误处理矩阵

## 13.1 登录错误

| HTTP 状态 | 当前处理 | 建议处理 |
|---|---|---|
| 400 | 显示 detail | 显示字段错误 |
| 401 | 显示 detail | 账号或密码错误 |
| 403 | 显示 detail | 授权无效/设备超限/账号停用 |
| 404 | 显示 detail | 项目不存在或服务器错误 |
| 429 | 显示 detail | 登录过于频繁 |
| 5xx | 网络请求失败或 detail | 服务器异常，请稍后 |

## 13.2 心跳错误

| HTTP 状态 | 当前处理 | 建议处理 |
|---|---|---|
| 200 | 上报成功 | 保持 |
| 401 | ensure_token | 刷新成功后可立即重试 |
| 403 | warning | 标记授权失效并提示 |
| 429 | warning | 降低频率 |
| 5xx | warning | 指数退避 |

## 13.3 参数错误

| HTTP 状态 | 当前处理 | 建议处理 |
|---|---|---|
| 200 | 解析参数 | 保持 |
| 401 | 返回空 | ensure_token 后重试 |
| 403 | 返回空 | 授权失效，退出或提示 |
| 422 | 返回空 | 打印参数格式错误 |
| 5xx | 返回空 | 使用上次缓存参数 |

## 13.4 热更新错误

| 场景 | 当前处理 | 建议处理 |
|---|---|---|
| 检查失败 | 跳过 | 非强制更新可跳过，强制更新应阻断 |
| 下载链接失败 | force_update 时 exit | 保持 |
| 下载失败 | return | force_update 时 exit |
| 校验失败 | 未见校验 | 必须拒绝安装 |
| 安装失败 | 未见捕获 | 记录、提示、必要时退出 |
| 版本写入 | 安装前写入 | 改为新版本启动后写入 |

---

## 十四、AndroidScript 侧待修订清单

## 14.1 P0：三端联调前必须处理

```text
□ 确认 Config.API_BASE_URL 清洗尾部斜杠。
□ 确认 Config.PROJECT_UUID 与 Verify 项目一致。
□ 确认 Verify.login 实机可用。
□ 确认 Heartbeat.start 实机可用。
□ 确认 CloudSync.pull_params 返回结构。
□ 确认 Updater.check_and_update 不会误写版本。
□ 确认 installLrPkg 在目标环境可用。
```

## 14.2 P1：联调后必须修订

```text
□ 心跳 401 刷新后立即重试。
□ CloudSync 增加 401 ensure_token 重试。
□ Updater 增加 401 ensure_token 重试。
□ 热更新增加 SHA-256 校验。
□ 强制更新失败时阻断继续运行。
□ 本地版本号改为新包启动后写入。
□ 设备指纹持久化。
```

## 14.3 P2：长期治理

```text
□ IMSI 是否上传做成配置开关。
□ game_data 按游戏适配文档规范化。
□ 参数响应结构固定。
□ Token 和密码本地存储策略审查。
□ 日志脱敏。
□ 失败重试和指数退避。
```

---

## 十五、AndroidScript 联调测试清单

## 15.1 登录测试

```text
□ 正确账号密码登录成功。
□ 密码错误登录失败。
□ 项目 UUID 错误登录失败。
□ 无授权登录失败。
□ 授权过期登录失败。
□ 设备超限登录失败。
□ 登录成功后 KV 保存 access_token。
□ 登录成功后 KV 保存 refresh_token。
□ 登录成功后写入 last_login_time。
```

## 15.2 Token 测试

```text
□ access_token 过期后 refresh 成功。
□ refresh_token 失效后重新登录成功。
□ 设备指纹变化时密码解密失败可提示重新登录。
□ logout 后本地 token 清空。
```

## 15.3 心跳测试

```text
□ 登录成功后启动心跳线程。
□ 心跳间隔约 30 秒。
□ 心跳体包含 device_fingerprint。
□ 心跳体包含 status。
□ 心跳体包含 game_data。
□ Verify Redis 中出现在线态。
□ Celery 落库后 device_runtime 有记录。
□ PCControl 能看到该设备。
□ 401 时尝试刷新 Token。
□ 429 时不会高频重试。
```

## 15.4 参数测试

```text
□ /api/params/get 返回参数。
□ AndroidScript 能解析 bool。
□ AndroidScript 能解析 int。
□ AndroidScript 能解析 string。
□ 参数为空时 game_main 能使用默认值。
□ 参数拉取失败时不崩溃。
```

## 15.5 热更新测试

```text
□ 当前版本 1.0.0，无新版本时不更新。
□ 后台上传 1.0.1 后能检测到更新。
□ 能获取 download_url。
□ 能下载 update.lrj。
□ 下载失败时有提示。
□ installLrPkg 成功。
□ restartScript 成功。
□ 新包启动后版本变为 1.0.1。
□ force_update=true 且下载失败时阻断运行。
□ SHA-256 校验逻辑补齐后测试通过。
```

---

## 十六、与 Verify 后端的契约差异待确认

## 16.1 热更新字段

AndroidScript 当前使用：

```text
check.latest_version
check.update_message
check.force_update
dl.download_url
```

Verify 后端必须确认是否返回这些字段。

待确认项：

```text
□ 后端 check_update 返回 latest_version 还是 version。
□ 后端 check_update 返回 update_message 还是 release_notes。
□ 后端 download_update 返回 download_url。
□ 后端是否返回 checksum_sha256。
```

## 16.2 参数字段

AndroidScript 当前兼容：

```text
data.params or data
```

Verify 后端应固定为：

```json
{
  "params": {}
}
```

待确认：

```text
□ 后端是否固定返回 params。
□ 参数值类型是否统一为 string。
□ bool 是否返回 true/false 还是 "true"/"false"。
```

## 16.3 登录字段

AndroidScript 当前期待：

```text
access_token
refresh_token
user_level
```

待确认：

```text
□ 后端是否返回 user_level。
□ 后端是否返回授权到期时间。
□ 后端是否返回 authorized_devices。
□ 后端是否返回 project 信息。
```

---

## 十七、建议新增到决策日志

```markdown
## D021 — AndroidScript 与 Verify 的接口调用边界

**日期：** 2026-04-29  
**状态：** 建议确认  
**关联文档：** [[03-安卓脚本框架/Verify接口调用清单]], [[01-网络验证系统/API路由清单]]

### 背景

AndroidScript 当前已经接入 Verify 登录、心跳、参数、热更新等核心接口，但部分错误处理、热更新校验、设备指纹持久化仍需治理。

### 决策

AndroidScript 与 Verify 的边界统一为：

```text
AndroidScript 负责:
  登录、心跳、拉参数、检查更新、下载并安装脚本包。

Verify 负责:
  身份认证、授权校验、设备绑定、心跳缓存与落库、参数存储、热更新元数据与下载链接。
```text

AndroidScript 不负责：

```text
1. 创建用户。
2. 管理代理。
3. 修改授权。
4. 上传热更新包。
5. 处理点数计费。
```text

### 理由

```text
1. 保持安卓端脚本轻量。
2. 避免在脚本端混入管理能力。
3. 减少安全风险。
4. 符合 Verify 作为平台中枢的定位。
```text

### 后续动作

```text
□ 补设备指纹持久化。
□ 补热更新 SHA-256 校验。
□ 补强制更新失败阻断。
□ 补 401 自动刷新后重试。
□ 补 AndroidScript 三端联调报告。
```text
```

---

## 十八、结论

当前 AndroidScript 已经具备接入 Verify 的核心代码基础。

已确认源码中已经实现：

```text
1. 登录 Verify。
2. 保存 Access Token / Refresh Token。
3. Refresh Token 自动刷新。
4. 安卓设备指纹生成。
5. 心跳上报。
6. 云端参数拉取。
7. 云端参数保存函数。
8. 安卓脚本热更新检查。
9. 安卓脚本热更新下载。
10. installLrPkg 安装更新包。
```

但当前不能标记为“安卓端 Verify 链路已稳定”。

当前最准确状态：

```text
AndroidScript Verify 接口调用源码已成型；
下一步必须进入实机联调；
热更新校验、设备指纹持久化、401 重试、强制更新失败处理、IMSI 合规策略仍需修订。
```

当前下一步：

```text
1. 使用真实 Verify base_url 和 project_uuid 启动 AndroidScript。
2. 验证登录。
3. 验证心跳。
4. 验证参数拉取。
5. 验证热更新。
6. 生成 AndroidScript 联调报告。
```