---
文件位置: 02-PC中控框架/Verify接口调用清单.md
名称: PCControl Verify 接口调用清单
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-29
版本: V1.0.0
状态: 草稿
关联文档:
  - "[[项目总大纲]]"
  - "[[01-网络验证系统/架构设计]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/三端联调测试清单]]"
  - "[[01-网络验证系统/前后端接口对照表]]"
  - "[[01-网络验证系统/风险与待决策清单]]"
  - "[[02-PC中控框架/架构设计]]"
  - "[[02-PC中控框架/PC安卓局域网通信方案]]"
  - "[[03-安卓脚本框架/Verify接口调用清单]]"
  - "[[编码规范]]"
  - "[[编码规范_补充_Markdown文档规范]]"
变更记录:
  - V1.0.0: 基于 HiveGreatSage-PCControl 当前源码反向整理 PC 中控调用 Verify 的接口、线程模型、字段契约、风险点与联调清单
---

# PCControl Verify 接口调用清单

← 返回 [[项目总大纲]] | 父节点: [[02-PC中控框架/架构设计]]

## 一、文档目的

本文用于沉淀 `HiveGreatSage-PCControl` 当前 PC 中控框架与 `HiveGreatSage-Verify` 网络验证系统之间的接口调用关系。

本文不是 Verify API 总表，而是从 **PC 中控调用方视角** 反向整理：

```text
1. PCControl 当前调用了哪些 Verify 接口。
2. 每个接口由哪个 Python 模块调用。
3. 每个接口依赖哪些配置项。
4. 每个接口的请求字段是什么。
5. 当前源码已经实现什么。
6. 当前源码还缺什么。
7. 哪些问题会影响三端联调。
8. 后续应该如何测试和修订。
```

---

## 二、当前结论

### 2.1 已确认

当前 PCControl 已经接入 Verify 的核心链路：

```text
认证登录:
  core/api_client/auth_api.py
  core/auth/auth_manager.py
  POST /api/auth/login

Token 刷新:
  core/api_client/auth_api.py
  core/auth/auth_manager.py
  POST /api/auth/refresh

登出:
  core/api_client/auth_api.py
  core/auth/auth_manager.py
  POST /api/auth/logout

当前用户信息:
  core/api_client/auth_api.py
  GET /api/auth/me

设备列表:
  core/api_client/device_api.py
  core/device/device_manager.py
  core/sync/sync_worker.py
  GET /api/device/list

单设备数据:
  core/api_client/device_api.py
  GET /api/device/data?device_fingerprint=...

脚本参数拉取:
  core/api_client/params_api.py
  GET /api/params/get

脚本参数保存:
  core/api_client/params_api.py
  POST /api/params/set

热更新检查:
  core/api_client/update_api.py
  core/updater/update_checker.py
  GET /api/update/check?client_type=pc&current_version=...

热更新下载:
  core/api_client/update_api.py
  core/updater/update_downloader.py
  GET /api/update/download?client_type=pc
```

### 2.2 已确认的线程边界

当前 `BaseClient` 是同步 `httpx` 客户端。

源码中已经明确：

```text
同步 HTTP 客户端必须在 QThread 中调用。
禁止在 UI 主线程直接调用。
```

当前已使用 QThread 的模块：

```text
1. 登录窗口登录线程。
2. SyncWorker 设备同步线程。
3. UpdateCheckWorker 热更新检查线程。
4. UpdateDownloadWorker 热更新下载线程。
```

### 2.3 待确认

以下内容当前不能写成已完成：

```text
1. PCControl 是否已经在真实 Verify 环境登录成功。
2. PC 登录是否确认不占安卓设备名额。
3. PCControl 是否能拉到 AndroidScript 心跳设备。
4. PCControl 是否能正确展示 device_runtime 中的 game_data。
5. PCControl 是否已经把参数编辑 UI 接到 /api/params/set。
6. PCControl 是否能正确解析 /api/params/get 的 params 列表。
7. PCControl 是否能真实下载 PC 更新包。
8. PCControl 打包为 exe 后，更新安装是否可用。
9. SyncWorker 的开发模式 mock fallback 是否会掩盖真实接口失败。
10. config/local.yaml、device_id.txt、last_login.json、logs 是否已从仓库治理。
```

---

## 三、相关源码文件

## 3.1 API 基础客户端

```text
core/api_client/base_client.py
```

职责：

```text
1. 保存 base_url。
2. 保存 timeout。
3. 注入 Bearer Token。
4. 统一发起 HTTP 请求。
5. 统一处理非 2xx 错误。
6. 抛出 ApiError。
7. 封装 get/post/put/delete。
```

当前特征：

```text
1. 使用同步 httpx.request。
2. 默认 Content-Type: application/json。
3. Token 通过 set_token 注入。
4. base_url 会 rstrip("/")。
5. 非 2xx 时尝试解析 detail 和 error_code。
6. 返回 JSON dict，非 JSON 返回空 dict。
```

---

## 3.2 认证 API

```text
core/api_client/auth_api.py
```

职责：

```text
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout
GET  /api/auth/me
```

---

## 3.3 认证管理器

```text
core/auth/auth_manager.py
```

职责：

```text
1. 构造登录 payload。
2. 调用 AuthApi.login。
3. 保存 Access Token。
4. 保存 Refresh Token。
5. 保存最后登录用户名。
6. 支持记住密码。
7. 支持 Refresh Token 刷新 Access Token。
8. 支持登出。
9. 将 ApiError 映射为用户友好错误。
```

---

## 3.4 设备 API

```text
core/api_client/device_api.py
core/device/device_manager.py
```

职责：

```text
GET /api/device/list
GET /api/device/data
```

`DeviceManager` 还负责：

```text
1. 调用 DeviceApi 拉取设备列表。
2. 合并本地设备元数据。
3. 读取和写入 config/device_meta.json。
4. 返回 DeviceInfo 列表给 UI。
```

---

## 3.5 同步线程

```text
core/sync/sync_worker.py
core/sync/sync_manager.py
```

职责：

```text
1. 每隔 sync.interval 秒拉取 Verify 设备列表。
2. 在 QThread 中执行网络请求。
3. 拉取成功后 emit devices_updated。
4. 401 时刷新 Token。
5. Token 刷新成功后立即重试一次。
6. Token 刷新失败后 emit token_expired。
7. 开发模式下可降级为模拟设备。
```

---

## 3.6 参数 API

```text
core/api_client/params_api.py
```

职责：

```text
GET  /api/params/get
POST /api/params/set
```

当前契约：

```text
get_params():
  返回 game_project_code, params, total。

set_params(params):
  params 是 list[dict]。
  每项格式:
    {"param_key": str, "param_value": str}
```

---

## 3.7 热更新 API

```text
core/api_client/update_api.py
core/updater/update_checker.py
core/updater/update_downloader.py
core/updater/update_installer.py
```

职责：

```text
1. 检查 PC 端是否有新版本。
2. 获取下载链接。
3. 下载 zip 更新包。
4. 校验 SHA-256。
5. 开发模式直接解压覆盖。
6. 打包 exe 模式生成 updater.bat 辅助更新。
```

---

## 3.8 配置系统

```text
core/utils/config.py
config/default.yaml
config/local.yaml
game/game_config.py
```

职责：

```text
1. default.yaml 提供默认配置。
2. local.yaml 覆盖本机配置。
3. Config.get("server.api_base_url") 读取 Verify 地址。
4. Config.get("server.project_uuid") 读取项目 UUID。
5. game/game_config.py 作为 fork 后的游戏元信息声明。
```

---

## 四、Verify 配置契约

## 4.1 `server.api_base_url`

### 当前位置

```yaml
server:
  api_base_url: "http://127.0.0.1:8000"
```

### 当前用途

被以下模块读取：

```text
AuthManager
DeviceManager
UpdateCheckWorker
UpdateDownloadWorker
```

### 设计要求

```text
1. 开发环境指向 http://127.0.0.1:8000。
2. 生产环境指向 https://正式域名。
3. 不建议使用 localhost，避免 IPv6 / IPv4 / 模拟器访问差异。
4. 不允许业务代码里写死 Verify 地址。
5. base_url 末尾可以有 /，BaseClient 会 rstrip("/")。
```

### 风险点

```text
config/local.yaml 当前属于本地环境配置，不应入库。
```

建议方案：

```text
1. 提交 config/local.example.yaml。
2. config/local.yaml 加入 .gitignore。
3. 每台机器自行复制 local.example.yaml。
```

---

## 4.2 `server.project_uuid`

### 当前位置

```yaml
server:
  project_uuid: "a46e3eff-2b46-4cb9-b58f-877917bf5c03"
```

### 当前用途

登录 payload 中：

```python
"project_uuid": self._config.get("server.project_uuid", "")
```

### 设计要求

```text
1. 必须与 Verify 管理后台创建的 game_project.project_uuid 一致。
2. 必须与 AndroidScript 的 Config.PROJECT_UUID 一致。
3. fork 新游戏时必须修改。
4. PCControl 不应让普通用户修改 project_uuid。
```

### 当前风险

当前 `game/game_config.py` 中的 `PROJECT_UUID` 与 `config/local.yaml` 中的 `server.project_uuid` 可能不一致。

已确认源码注释中写明：

```text
运行时实际从 Config.get("server.project_uuid") 读取；
game/game_config.py 仅作为代码可读性声明。
```

建议：

```text
1. 新游戏 fork checklist 中加入 project_uuid 双处同步。
2. 或取消 game/game_config.py 中的 PROJECT_UUID 常量，避免双真相源。
3. 保留时必须在启动日志中打印两者并检查一致。
```

---

## 4.3 `sync.interval`

### 当前位置

```yaml
sync:
  interval: 10
```

### 当前用途

`SyncManager` 创建 `SyncWorker` 时读取：

```python
interval = int(config.get("sync.interval", 10))
```

### 设计要求

```text
1. PCControl 拉取设备列表默认 10 秒一次。
2. 该频率应与 Verify 压测目标匹配。
3. 多 PC 同时登录时要评估查询压力。
```

### 风险点

```text
如果用户数多，PC 同步间隔过短，会增加 Verify 查询压力。
```

建议：

```text
1. 默认 10 秒。
2. 大客户版本可配置 10-30 秒。
3. UI 上不暴露过低值。
```

---

## 4.4 `update.check_on_startup`

### 当前位置

```yaml
update:
  check_on_startup: true
```

### 当前用途

用于启动后检查 PC 中控更新。

待确认：

```text
当前 Application 是否已经读取该配置并自动启动 UpdateCheckWorker。
```

---

## 4.5 `APP_VERSION`

### 当前位置

```text
core/utils/constants.py
```

### 当前用途

`UpdateCheckWorker` 使用：

```python
from core.utils.constants import APP_VERSION
api.check(current_version=APP_VERSION, client_type="pc")
```

### 设计要求

```text
1. APP_VERSION 必须与打包版本一致。
2. Verify 热更新版本号必须使用 MAJOR.MINOR.PATCH。
3. 每次打包发布必须同步 APP_VERSION。
```

---

## 五、认证接口调用

## 5.1 登录接口

### Python 模块

```text
core/api_client/auth_api.py
core/auth/auth_manager.py
```

### 函数

```python
AuthApi.login(payload)
AuthManager.login(username, password, remember_password=False)
```

### Verify 接口

```text
POST /api/auth/login
```

### 当前请求体

```json
{
  "username": "user001",
  "password": "User@2026!",
  "project_uuid": "项目 UUID",
  "device_fingerprint": "pc_control_uuid",
  "client_type": "pc"
}
```

### 当前源码构造

```python
{
    "username": username,
    "password": password,
    "project_uuid": self._config.get("server.project_uuid", ""),
    "device_fingerprint": hardware_serial,
    "client_type": CLIENT_TYPE,
}
```

其中：

```text
CLIENT_TYPE = pc
hardware_serial 来自 config/device_id.txt。
首次运行时生成 UUID4。
```

### 成功处理

登录成功后：

```text
1. 保存 access_token 到内存。
2. AuthApi.set_token(access_token)。
3. 解析 UserInfo。
4. Refresh Token 加密后存入 keyring。
5. 如果勾选记住密码，明文密码存入 keyring。
6. 保存最后登录用户名到 config/last_login.json。
```

### 错误映射

当前 `AuthManager._map_api_error()` 已映射：

```text
401 → 用户名或密码错误 / INVALID_CREDENTIALS
403 → 账号无权限或已过期 / FORBIDDEN
404 → 用户不存在 / USER_NOT_FOUND
422 → 请求格式错误 / INVALID_PAYLOAD
429 → 请求过于频繁 / RATE_LIMIT
其他 → SERVER_ERROR
```

### 风险点

```text
1. 404 可能不只是用户不存在，也可能是项目 UUID 不存在。
2. UserInfo 只解析 user_level 和 game_project_code，未解析授权到期、设备额度等字段。
3. PC 登录是否不占设备名额，需要 Verify 后端实测确认。
4. keyring 记住密码属于用户便利功能，但需确认安全边界。
```

### 建议方案

```text
1. 登录成功响应建议补齐 valid_until、authorized_devices、project_name。
2. PCControl 登录后显示项目名、等级、到期时间。
3. 404 文案改为“用户或项目不存在”更稳妥。
4. 记住密码默认不勾选。
```

---

## 5.2 Refresh Token 接口

### Python 模块

```text
core/api_client/auth_api.py
core/auth/auth_manager.py
```

### 函数

```python
AuthApi.refresh_token(refresh_token)
AuthManager.refresh_access_token()
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
1. 从 keyring 读取加密 Refresh Token。
2. 使用 device_id 解密。
3. 调用 /api/auth/refresh。
4. 更新 access_token。
5. 如果后端返回新的 refresh_token，则更新 keyring。
6. 返回 True / False。
```

### 当前优点

```text
已兼容 Refresh Token 轮换。
```

### 风险点

```text
1. device_id.txt 如果丢失，会导致 Refresh Token 无法解密。
2. Refresh Token 读取失败后只能重新登录。
3. device_id.txt 当前属于本机运行态文件，不应入库。
```

---

## 5.3 登出接口

### Python 模块

```text
core/api_client/auth_api.py
core/auth/auth_manager.py
```

### 函数

```python
AuthApi.logout()
AuthManager.logout()
```

### Verify 接口

```text
POST /api/auth/logout
```

### 当前行为

```text
1. 如果 access_token 存在，调用 logout。
2. 不论接口是否成功，清空内存 access_token。
3. 清空 AuthApi Token。
4. 清空 UserInfo。
```

### 风险点

```text
1. AuthApi.logout 当前不发送 refresh_token。
2. Verify 后端如果要求 refresh_token 才能删除对应 Refresh Token，则 PC 端登出可能只清理 Access Token。
3. 当前不删除 keyring 中保存的 Refresh Token。
```

### 建议方案

```text
1. 确认 Verify logout 是否需要 refresh_token。
2. 如果需要，AuthApi.logout 应传 refresh_token。
3. 用户主动退出时可选择是否清除本机 Refresh Token 和记住密码。
```

---

## 5.4 当前用户信息接口

### Python 模块

```text
core/api_client/auth_api.py
```

### 函数

```python
AuthApi.me()
```

### Verify 接口

```text
GET /api/auth/me
```

### 当前状态

已封装，但待确认 PCControl 主流程是否实际调用。

### 建议用途

```text
1. 登录后确认 Token。
2. 刷新 Token 后重新拉用户授权信息。
3. 显示账号等级、项目、到期时间。
4. 检查账号被停用或授权变化。
```

---

## 六、设备数据接口调用

## 6.1 设备列表接口

### Python 模块

```text
core/api_client/device_api.py
core/device/device_manager.py
core/sync/sync_worker.py
```

### Verify 接口

```text
GET /api/device/list
```

### Header

```text
Authorization: Bearer {pc_access_token}
```

### 当前调用链

```text
SyncWorker._do_sync()
  → DeviceManager.fetch_devices()
  → DeviceApi.get_device_list()
  → GET /api/device/list
  → DeviceInfo.from_api(raw, meta)
  → emit devices_updated(devices)
  → UI 刷新设备表格
```

### 当前响应期待

```json
{
  "devices": [
    {
      "device_id": "android_device_001",
      "user_id": 1,
      "status": "running",
      "last_seen": "2026-04-29T00:00:00",
      "game_data": {},
      "is_online": true
    }
  ],
  "total": 1,
  "online_count": 1
}
```

### 本地合并字段

`DeviceManager` 会合并：

```text
config/device_meta.json
```

本地元数据结构：

```json
{
  "<fingerprint>": {
    "alias": "A-001",
    "role": "captain",
    "note": "",
    "activated": false
  }
}
```

### 风险点

```text
1. device_meta.json 是本地状态，不应入库。
2. Verify 返回 device_id，PCControl 本地称 fingerprint，命名需统一。
3. game_data 是自由 JSON，UI 展示需要游戏适配层。
4. 开发模式 mock fallback 可能掩盖真实接口失败。
```

### 建议方案

```text
1. 文档统一字段:
   Verify 字段叫 device_id。
   PC 本地内部可以叫 fingerprint，但 UI 和文档注明等价关系。

2. device_meta.json 加入 .gitignore。

3. mock fallback 仅开发模式可用，生产模式必须暴露同步错误。

4. 每个游戏定义 game_data 展示字段契约。
```

---

## 6.2 单设备数据接口

### Python 模块

```text
core/api_client/device_api.py
```

### 函数

```python
DeviceApi.get_device_data(device_fingerprint)
```

### Verify 接口

```text
GET /api/device/data?device_fingerprint=...
```

### 当前用途

已封装，待确认 UI 是否已经在设备详情、右键菜单、详情弹窗中调用。

### 返回期待

```json
{
  "device_id": "android_device_001",
  "user_id": 1,
  "status": "running",
  "last_seen": "2026-04-29T00:00:00",
  "game_data": {},
  "is_online": true,
  "source": "redis"
}
```

### 风险点

```text
1. source 可能是 redis / database / not_found。
2. UI 应区分实时数据和历史数据。
3. 如果 Redis 过期但 database 有旧数据，不能误判为在线。
```

---

## 七、SyncWorker 同步机制

## 7.1 同步频率

当前默认：

```text
10 秒一次
```

配置：

```yaml
sync:
  interval: 10
```

### 当前流程

```text
1. 登录成功后启动 SyncManager。
2. SyncManager 创建 SyncWorker。
3. SyncWorker 每 10 秒拉 /api/device/list。
4. 成功后 emit devices_updated。
5. UI 主线程收到信号后刷新表格。
```

### 7.2 401 处理

当前流程：

```text
1. fetch_devices 抛出 status_code=401。
2. SyncWorker 调用 auth_manager.refresh_access_token()。
3. 刷新成功后立即重试一次 fetch。
4. 刷新后仍 401，emit token_expired。
5. 刷新失败，emit token_expired。
```

### 当前优点

```text
1. 避免 UI 主线程阻塞。
2. 401 自动恢复设计合理。
3. 已规避 ApiError 类身份问题，改为 getattr(status_code) 检测。
```

### 7.3 风险点

```text
1. 仅设备列表同步有 401 自动刷新。
2. 参数保存、热更新检查是否也需要统一 401 刷新重试，待确认。
3. mock fallback 只应开发使用。
4. token_expired 后 UI 是否停止 SyncWorker，需要联调确认。
```

### 建议方案

```text
1. 提取统一 AuthenticatedApiRunner。
2. 所有 User Token 接口遇到 401 都先 refresh 一次再重试。
3. refresh 失败统一通知重新登录。
```

---

## 八、脚本参数接口调用

## 8.1 拉取参数接口

### Python 模块

```text
core/api_client/params_api.py
```

### 函数

```python
ParamsApi.get_params()
```

### Verify 接口

```text
GET /api/params/get
```

### 当前响应契约

Verify 当前参数响应结构应为：

```json
{
  "game_project_code": "yeya",
  "params": [
    {
      "param_key": "auto_task",
      "param_type": "bool",
      "value": "true",
      "is_default": true,
      "display_name": "自动任务",
      "description": "是否启用自动任务",
      "options": null,
      "sort_order": 1
    }
  ],
  "total": 1
}
```

### 当前用途

已封装，待确认 `game/params_widget.py` 或主窗口参数页是否实际接入。

### 风险点

```text
1. PCControl 参数 UI 当前可能仍是空骨架。
2. params 是 list[ParamItem]，不是 dict。
3. value 使用 JSON 字符串格式，UI 需要按 param_type 解析。
4. enum 需要使用 options 生成下拉框。
```

---

## 8.2 保存参数接口

### Python 模块

```text
core/api_client/params_api.py
```

### 函数

```python
ParamsApi.set_params(params)
```

### Verify 接口

```text
POST /api/params/set
```

### 当前请求体

```json
{
  "params": [
    {
      "param_key": "auto_task",
      "param_value": "true"
    },
    {
      "param_key": "target_map",
      "param_value": "\"东海岸\""
    }
  ]
}
```

### 当前响应契约

```json
{
  "game_project_code": "yeya",
  "updated_count": 2,
  "failed_count": 0,
  "results": [
    {
      "param_key": "auto_task",
      "success": true,
      "error": null
    }
  ]
}
```

### 风险点

```text
1. 参数值统一使用 JSON 字符串，不是 Python 原生 bool/int/string。
2. string 参数需要传 "\"东海岸\"" 还是 "东海岸"，必须和 Verify 当前校验保持一致。
3. failed_count > 0 时，UI 不能简单提示“全部成功”。
4. AndroidScript 拉参数时也要能解析同一格式。
```

### 建议方案

```text
1. 参数 UI 层增加 ParamCodec。
2. ParamCodec 负责 Python 值 ↔ JSON 字符串转换。
3. 保存后展示每项 result。
4. failed_count > 0 时高亮失败参数。
```

---

## 九、热更新接口调用

## 9.1 检查更新

### Python 模块

```text
core/api_client/update_api.py
core/updater/update_checker.py
```

### Verify 接口

```text
GET /api/update/check?client_type=pc&current_version={APP_VERSION}
```

### 当前函数

```python
UpdateApi.check(current_version, client_type="pc")
```

### 当前响应期待

```json
{
  "need_update": true,
  "current_version": "1.0.0",
  "client_type": "pc",
  "game_project_code": "yeya",
  "force_update": false,
  "release_notes": "更新说明",
  "checksum_sha256": "..."
}
```

### 当前解析

`UpdateCheckWorker` 会构造：

```python
UpdateInfo(
    need_update=True,
    force_update=data.get("force_update", False),
    new_version=data.get("latest_version") or data.get("version") or "",
    current_version=APP_VERSION,
    release_notes=data.get("release_notes", ""),
    checksum_sha256=data.get("checksum_sha256", ""),
)
```

待确认：

```text
当前源码是否同时兼容 latest_version / version。
如果没有，需要补兼容或统一 Verify 响应字段。
```

---

## 9.2 下载更新

### Python 模块

```text
core/api_client/update_api.py
core/updater/update_downloader.py
```

### Verify 接口

```text
GET /api/update/download?client_type=pc
```

### 当前函数

```python
UpdateApi.get_download_url(client_type="pc")
```

### 当前响应期待

```json
{
  "download_url": "https://...",
  "expires_at": "2026-04-29T00:00:00",
  "version": "1.0.1",
  "checksum_sha256": "..."
}
```

### 当前下载行为

```text
1. 调用 Verify 获取 download_url。
2. 使用 httpx.stream 下载文件。
3. 下载到系统临时目录:
   hgs_update_{new_version}.zip
4. 下载过程中计算 SHA-256。
5. 如果 expected_sha 存在，则校验。
6. 校验失败删除临时文件。
7. 下载成功 emit finished(zip_path)。
```

### 当前优点

```text
PCControl 已经实现 SHA-256 校验。
```

---

## 9.3 安装更新

### Python 模块

```text
core/updater/update_installer.py
```

### 当前策略

开发模式：

```text
1. 解压 zip 到项目根目录。
2. 删除 zip。
3. os.execv 重启 Python 进程。
```

打包模式：

```text
1. 生成 updater.bat。
2. 主程序退出。
3. bat 等待 2 秒。
4. powershell Expand-Archive 覆盖应用目录。
5. 删除 zip。
6. 启动 exe。
7. 删除 bat 自身。
```

### 风险点

```text
1. .exe 运行时自身文件被占用，必须依赖外部 updater.bat。
2. 更新包结构必须与应用目录一致。
3. 如果解压失败，可能造成半更新状态。
4. 当前未见回滚机制。
5. updater.bat 使用 gbk 写入，中文路径需实测。
6. 杀毒软件可能拦截 bat 或覆盖 exe。
```

### 建议方案

```text
1. PC 更新包使用固定结构。
2. 更新前备份当前版本。
3. 解压到临时目录后再替换。
4. 增加 rollback.bat 或保留 last_good。
5. 打包后必须在真实 exe 环境测试。
```

---

## 十、当前未调用或待确认接口

## 10.1 `/api/auth/revoke-all`

当前 PCControl 未确认调用。

建议用途：

```text
1. 用户点击“踢出所有设备”。
2. PCControl 触发 Verify revoke-all。
3. AndroidScript 下次心跳或刷新 Token 时失效。
```

风险：

```text
Verify 当前 revoke-all 可能无法让全部已签发 Access Token 即时失效。
```

建议：

```text
暂不在 UI 中承诺“立即踢下线所有设备”。
```

---

## 10.2 `/api/auth/me`

已封装，待确认主流程是否调用。

建议：

```text
登录成功后调用一次 /me，作为服务端最终用户状态。
```

---

## 10.3 `/api/device/heartbeat`

PCControl 不应调用。

说明：

```text
心跳由 AndroidScript 调用。
PCControl 只读取设备列表和设备详情。
```

---

## 10.4 `/api/device/imsi`

PCControl 不应调用。

说明：

```text
IMSI 采集如启用，也应由 AndroidScript 上报。
PCControl 最多展示脱敏结果。
```

---

## 十一、字段契约总表

## 11.1 PCControl → Verify 登录字段

| 字段 | 当前来源 | 必填 | 说明 |
|---|---|---|---|
| `username` | 登录窗口输入 | 是 | 用户账号 |
| `password` | 登录窗口输入 | 是 | 用户密码 |
| `project_uuid` | `server.project_uuid` | 是 | 游戏项目 UUID |
| `device_fingerprint` | `config/device_id.txt` | 是 | PC 本机 UUID |
| `client_type` | `CLIENT_TYPE` | 是 | 固定 `pc` |

## 11.2 Verify → PCControl 登录响应字段

| 字段 | 当前使用 | 风险 |
|---|---|---|
| `access_token` | 必须 | 登录成功凭据 |
| `refresh_token` | 必须 | 自动刷新凭据 |
| `username` | 使用 | 显示账号 |
| `user_level` | 使用 | 显示等级 |
| `game_project_code` | 使用 | 填入 UserInfo.game_name |
| `valid_until` | 未使用 | 建议补 |
| `authorized_devices` | 未使用 | 建议补 |
| `project_name` | 未使用 | 建议补 |

## 11.3 Verify → PCControl 设备列表字段

| 字段 | 当前用途 | 说明 |
|---|---|---|
| `device_id` | 映射 DeviceInfo | 安卓设备指纹 |
| `user_id` | 可展示/关联 | 用户 ID |
| `status` | 展示运行状态 | running / idle / error |
| `last_seen` | 展示最后心跳 | ISO 时间 |
| `game_data` | 游戏数据展示 | JSON |
| `is_online` | 在线状态 | boolean |

## 11.4 PCControl → Verify 参数保存字段

| 字段 | 当前来源 | 说明 |
|---|---|---|
| `params` | 参数编辑 UI | list |
| `param_key` | 参数定义 | 参数键 |
| `param_value` | UI 编码后值 | JSON 字符串 |

## 11.5 Verify → PCControl 参数响应字段

| 字段 | 说明 |
|---|---|
| `game_project_code` | 当前项目 |
| `params` | 参数列表 |
| `total` | 参数数量 |

单参数：

| 字段 | 说明 |
|---|---|
| `param_key` | 参数键 |
| `param_type` | int / float / string / bool / enum |
| `value` | 当前生效值 |
| `is_default` | 是否默认值 |
| `display_name` | UI 显示名 |
| `description` | 参数说明 |
| `options` | enum 选项 |
| `sort_order` | 排序 |

## 11.6 Verify → PCControl 热更新字段

| 字段 | 当前用途 |
|---|---|
| `need_update` | 是否有更新 |
| `force_update` | 是否强制 |
| `version` / `latest_version` | 新版本号 |
| `release_notes` | 更新说明 |
| `checksum_sha256` | 下载校验 |
| `download_url` | 下载地址 |
| `expires_at` | 下载链接过期时间 |

---

## 十二、错误处理矩阵

## 12.1 登录错误

| HTTP 状态 | 当前处理 | 建议处理 |
|---|---|---|
| 401 | 用户名或密码错误 | 保持 |
| 403 | 账号无权限或已过期 | 细分授权过期、设备超限、账号停用 |
| 404 | 用户不存在 | 改为用户或项目不存在 |
| 422 | 请求格式错误 | 提示联系管理员 |
| 429 | 请求过于频繁 | 保持 |
| timeout | 连接服务器超时 | 保持 |
| network | 无法连接服务器 | 保持 |

---

## 12.2 设备同步错误

| 场景 | 当前处理 | 建议处理 |
|---|---|---|
| 200 | emit devices_updated | 保持 |
| 401 | refresh token 后重试 | 保持 |
| refresh 失败 | emit token_expired | 保持 |
| 403 | emit sync_error | 提示授权失效 |
| 429 | emit sync_error | 拉长同步间隔 |
| 5xx | emit sync_error 或 mock | 生产禁止 mock |
| 网络错误 | 开发模式 mock fallback | 生产直接显示错误 |

---

## 12.3 参数错误

| 场景 | 当前处理 | 建议处理 |
|---|---|---|
| GET 成功 | 返回 dict | UI 渲染 |
| SET 成功 | 返回结果明细 | 显示成功/失败数量 |
| failed_count > 0 | 待确认 | 高亮失败项 |
| 401 | 待确认 | refresh 后重试 |
| 422 | 待确认 | 显示参数校验错误 |

---

## 12.4 热更新错误

| 场景 | 当前处理 | 建议处理 |
|---|---|---|
| 检查失败 | 跳过并发送 check_failed | 非强制更新可跳过 |
| 下载地址失败 | failed | 显示错误 |
| 下载失败 | failed | 保持 |
| SHA-256 失败 | 删除文件并 failed | 保持 |
| 安装失败 | 记录异常 | UI 需要明确提示 |
| exe 更新失败 | 可能需 bat | 必须实测 |

---

## 十三、PCControl 侧待修订清单

## 13.1 P0：三端联调前必须确认

```text
□ server.api_base_url 指向当前 Verify。
□ server.project_uuid 与 Verify 项目一致。
□ server.project_uuid 与 AndroidScript PROJECT_UUID 一致。
□ PC 登录 Verify 成功。
□ PC 登录不占安卓设备名额。
□ SyncWorker 能拉 /api/device/list。
□ AndroidScript 心跳后 PC 能看到设备。
□ 401 Token 刷新链路可用。
□ config/local.yaml 不作为代码真相源。
```

---

## 13.2 P1：联调后必须修订

```text
□ 登录成功后调用 /api/auth/me 获取完整用户状态。
□ UserInfo 补 valid_until、authorized_devices、project_name。
□ 参数 UI 接入 /api/params/get 和 /api/params/set。
□ 增加 ParamCodec 处理 JSON 字符串值。
□ 热更新字段统一 latest_version / version。
□ 打包 exe 热更新实测 updater.bat。
□ mock fallback 增加明显“开发模式”标识。
```

---

## 13.3 P2：长期治理

```text
□ 引入统一 401 refresh retry 机制。
□ device_meta.json 纳入本地配置治理。
□ last_login.json 纳入本地配置治理。
□ device_id.txt 纳入本地配置治理。
□ logs/ 纳入仓库清理。
□ local.yaml 改为 local.example.yaml。
□ 添加 PCControl 接口集成测试。
```

---

## 十四、PCControl 联调测试清单

## 14.1 登录测试

```text
□ 正确账号密码登录成功。
□ 密码错误登录失败。
□ 项目 UUID 错误登录失败。
□ 无授权登录失败。
□ 授权过期登录失败。
□ 用户停用登录失败。
□ 登录限流显示友好提示。
□ 登录成功后 access_token 存在。
□ 登录成功后 refresh_token 存入 keyring。
□ 最后登录用户名写入 last_login.json。
```

---

## 14.2 Token 测试

```text
□ Access Token 过期后，SyncWorker 刷新成功。
□ Refresh Token 失效后，弹出重新登录。
□ device_id.txt 删除后，旧 Refresh Token 无法解密，能提示重新登录。
□ 登出后内存 access_token 清空。
□ 登出后 SyncWorker 停止。
```

---

## 14.3 设备同步测试

```text
□ AndroidScript 登录并上报心跳。
□ PCControl 登录同一用户同一项目。
□ SyncWorker 启动。
□ /api/device/list 返回设备。
□ UI 显示设备在线。
□ UI 显示 last_seen。
□ UI 显示 status。
□ UI 可展示 game_data。
□ AndroidScript 停止心跳后，PC 端在线态变为离线。
□ 网络断开时 UI 显示同步错误。
```

---

## 14.4 参数测试

```text
□ /api/params/get 能返回参数定义。
□ bool 参数渲染为复选框。
□ int 参数渲染为数字输入。
□ string 参数渲染为文本输入。
□ enum 参数渲染为下拉框。
□ 修改参数后 POST /api/params/set。
□ 保存成功后提示 updated_count。
□ failed_count > 0 时显示失败项。
□ AndroidScript 再次拉参数能拿到新值。
```

---

## 14.5 热更新测试

```text
□ 当前 APP_VERSION = 1.0.0。
□ Verify 上传 pc 1.0.1 包。
□ PCControl 检查到新版本。
□ 弹窗显示 release_notes。
□ 下载 zip 包。
□ SHA-256 校验通过。
□ 开发模式解压重启成功。
□ exe 模式生成 updater.bat。
□ exe 模式覆盖更新成功。
□ force_update=true 时不能忽略更新。
```

---

## 14.6 仓库治理测试

```text
□ config/local.yaml 不入库。
□ config/device_id.txt 不入库。
□ config/last_login.json 不入库。
□ config/device_meta.json 不入库。
□ logs/ 不入库。
□ .idea/ 不入库。
□ 提供 config/local.example.yaml。
```

---

## 十五、与 Verify 后端的契约差异待确认

## 15.1 登录响应字段不足

当前 PCControl 只解析：

```text
username
user_level
game_project_code
```

待确认 Verify 是否返回：

```text
valid_until
authorized_devices
project_name
project_uuid
```

建议：

```text
PCControl 登录后需要显示：
1. 用户名。
2. 当前项目。
3. 用户等级。
4. 授权到期。
5. 授权设备数。
```

---

## 15.2 参数值格式

Verify 当前说明：

```text
参数值统一使用 JSON 字符串格式。
```

例如：

```text
int:    "42"
float:  "3.14"
bool:   "true"
string: "\"北境\""
enum:   "1"
```

PCControl 参数 UI 必须确认：

```text
1. 保存 string 时是否要 json.dumps。
2. 展示 string 时是否要 json.loads。
3. bool 是否传 "true" / "false"。
```

---

## 15.3 热更新版本字段

PCControl 需要确认 Verify check 返回字段：

```text
version
latest_version
```

建议统一为：

```text
version
```

或：

```text
latest_version
```

二选一，不要长期双兼容。

---

## 15.4 PC 登录设备绑定口径

待确认：

```text
PC 登录是否写 device_binding？
PC 登录是否占 authorized_devices？
```

建议：

```text
PC 登录不占设备名额。
AndroidScript 登录占设备名额。
```

测试用例：

```text
authorized_devices = 1
安卓设备 A 登录成功
PC 登录成功
安卓设备 B 登录失败
```

---

## 十六、建议新增到决策日志

```markdown
## D022 — PCControl 与 Verify 的接口调用边界

**日期：** 2026-04-29  
**状态：** 建议确认  
**关联文档：** [[02-PC中控框架/Verify接口调用清单]], [[01-网络验证系统/API路由清单]]

### 背景

PCControl 当前已经接入 Verify 登录、设备列表、设备详情、脚本参数、热更新等核心接口。PCControl 是用户操作和设备监控端，不应直接访问数据库或绕过 Verify。

### 决策

PCControl 与 Verify 的边界统一为：

```text
PCControl 负责:
  登录、展示设备、修改参数、检查和下载 PC 更新包、本地 UI 与 ADB 辅助能力。

Verify 负责:
  身份认证、授权校验、设备绑定、设备运行数据、参数存储、热更新元数据和下载链接。
```

PCControl 不负责：

```text
1. 直接访问 PostgreSQL。
2. 直接访问 Redis。
3. 直接替 AndroidScript 上报心跳。
4. 管理代理点数。
5. 上传热更新包。
6. 处理项目准入审核。
```

### 理由

```text
1. 保持 PC 中控为客户端。
2. 避免客户端拥有过多管理权限。
3. 保证所有核心数据以 Verify 为真相源。
4. 支撑换 PC 登录同一账号仍能看到云端设备数据。
```

### 后续动作

```text
□ 补 PCControl 三端联调报告。
□ 补参数 UI 与 Verify 参数契约对齐。
□ 补 PC 热更新 exe 打包测试。
□ 补 PC 登录不占设备名额测试。
□ 清理 local.yaml、device_id.txt、last_login.json、logs。
```
```

---

## 十七、结论

当前 PCControl 已经具备接入 Verify 的核心代码基础。

已确认源码中已经实现：

```text
1. 登录 Verify。
2. 生成 PC 本机 device_fingerprint。
3. 保存 Access Token。
4. 加密保存 Refresh Token。
5. Refresh Token 自动刷新。
6. 设备列表定时同步。
7. 401 后刷新 Token 并重试。
8. 单设备详情接口封装。
9. 参数 GET/SET 接口封装。
10. PC 热更新检查。
11. PC 更新包下载。
12. SHA-256 校验。
13. 开发模式更新安装。
14. exe 模式 updater.bat 方案。
```

但当前不能标记为“PCControl Verify 链路已稳定”。

当前最准确状态：

```text
PCControl Verify 接口调用源码已成型；
下一步必须进入真实 Verify + AndroidScript 三端联调；
参数 UI、PC 热更新 exe 安装、本地运行态文件治理、PC 不占设备名额测试仍需补齐。
```

当前下一步：

```text
1. 使用真实 Verify base_url 和 project_uuid 启动 PCControl。
2. 验证登录。
3. 启动 AndroidScript 心跳。
4. 验证 PCControl 拉设备列表。
5. 验证参数保存与安卓拉取。
6. 验证 PC 热更新检查和下载。
7. 生成 PCControl 联调报告。
```