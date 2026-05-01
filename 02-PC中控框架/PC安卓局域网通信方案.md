---
文件位置: 02-PC中控框架/PC安卓局域网通信方案.md
名称: PC安卓局域网通信方案
作者: 蜂巢·大圣 (HiveGreatSage)
时间: 2026-05-01
版本: V1.0.0
状态: 草稿
关联文档:
  - "[[00-项目总控/项目总大纲]]"
变更记录:
  - V1.0.0: Obsidian 去漂移重构生成
---

r"""
文件位置: 02-PC中控框架/PC安卓局域网通信方案.md
名称: PC↔安卓局域网通信方案设计
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-16
版本: V1.0.1
功能及相关说明:
  确定 PC 中控与安卓脚本在局域网内的通信方案。
  基于懒人精灵函数文档实测能力推导，方案 A（手动 IP）为主，B/D 为后续扩展。

改进内容:
  V1.0.1 - 基于懒人精灵函数文档调整：仅客户端能力，PC 必须当服务端
  V1.0.0 - 初始版本

调试信息:
  已知问题: 无
"""

# PC ↔ 安卓局域网通信方案设计

> **文档性质：** 技术方案设计 + 实施指南
> **文档状态：** 草稿 v1（已基于懒人精灵实际能力锁定）
> **最后更新：** 2026-04-16
> **存放位置：** Obsidian → 02-PC中控框架/PC安卓局域网通信方案.md
> **前置文档：** [[项目总大纲]] v3, [[02-PC中控架构设计]], 懒人精灵函数文档
> **关联Claude对话：** 本次对话

---

## 一、关键技术事实（来自懒人精灵函数文档）

### 1.1 懒人精灵 Lua 的网络能力清单

| 函数 | 类型 | 说明 |
|------|------|------|
| `httpGet(url, [timeout], [header])` | HTTP 客户端 | 主动 GET 请求 |
| `httpPost(url, postdata, [timeout], [header])` | HTTP 客户端 | 主动 POST 请求 |
| `startWebSocket(url, onOpened, onClosed, onError, onRecv)` | WebSocket 客户端 | 主动连接 ws:// 服务器，注意 wss 暂不支持 |
| `sendWebSocket(handle, text)` | WebSocket 客户端 | 通过已建立的连接发送数据 |
| `uploadFile(url, uploadfile, [timeout])` | HTTP 客户端 | 上传文件 |
| `http.request(arg)` (luasocket) | HTTP 客户端 | 主动 HTTP 请求（支持 HTTPS） |
| `downloadFile(url, savepath)` | HTTP 客户端 | 下载文件 |

### 1.2 关键结论

```
✓ 懒人精灵能做的:
  · 主动发起 HTTP 请求
  · 主动连接 WebSocket 服务器
  · 主动上传/下载文件

✗ 懒人精灵做不到的:
  · 监听 TCP 端口（无 server socket API）
  · 监听 UDP 端口（无 udp.bind 类 API）
  · 接收主动推送的 HTTP 请求

⚠ 已知限制:
  · WebSocket 仅支持 ws://（不支持 wss://）
    → 局域网内问题不大，公网传输需通过 HTTP API 走云端
```

### 1.3 这个事实如何决定架构

```
传统的"PC 广播 → 安卓监听"方案在懒人精灵上不可行。
原因：安卓脚本无法监听任何端口。

正确的架构方向：
  PC = 服务端（WebSocket Server）  ← 监听端口，等待连接
  安卓 = 客户端（WebSocket Client）  ← 主动连接 PC

这恰好与方案 A（用户在脚本中填入 PC 的 IP）天然契合。
```

---

## 二、最终方案：手动 IP + WebSocket 双向通信

### 2.1 方案核心

```
角色分工:
  PC 中控:  WebSocket 服务端，监听 8889 端口
  安卓脚本: WebSocket 客户端，主动连接 PC

配对方式:
  · PC 中控启动后顶部状态栏显示「内网 IP : 端口 [复制]」按钮
  · 用户首次配置时把 IP 填入安卓脚本的配置（懒人精灵的脚本UI界面）
  · 配置存到设备本地 → 后续启动自动连
  · 工作室固定 IP 场景下「填一次，永久用」

数据流向:
  · 安卓启动 → startWebSocket → onOpened → 上报设备指纹+用户token认证
  · PC 收到认证 → 加入设备列表 → 推送当前任务/参数
  · 通信全程双向：PC 推送指令、安卓推送状态/结果
  · 断线 → 安卓 onClosed 触发重连（3 秒间隔）
```

### 2.2 端口约定

```
PC 中控固定端口：
  8889 — WebSocket 服务端口（局域网组队/状态推送）
  8890 — 备用端口（端口冲突时切换）

为什么用 8889 而不是常见的 80/8080：
  · 避开常见端口，减少与其他软件冲突
  · 8888 是常见调试端口避开
  · 8889 是 PC↔安卓约定，不需要对外公开
```

### 2.3 PC 中控的 IP 显示设计

```
位置：主窗口顶部状态栏，永久可见

UI 元素：
  ┌──────────────────────────────────────────────────────────────┐
  │  内网IP: 192.168.1.100:8889  [复制]   状态: ● 监听中         │
  └──────────────────────────────────────────────────────────────┘

行为细节：
  · 启动时自动检测本机所有内网 IP（排除 127.0.0.1 和虚拟网卡）
  · 多个网卡时 → 默认显示第一个，下拉可切换
  · 检测到 IP 后立即在 8889 端口启动 WebSocket Server
  · 端口被占用 → 自动尝试 8890 → 仍失败提示用户
  · [复制] 按钮一键复制完整地址（含端口）到剪贴板
  · 已连接设备数实时显示，例如「已连接 23/100 台」
```

---

## 三、PC 中控实现要点（PySide 6）

### 3.1 内网 IP 检测

```python
r'''
文件位置: core/team/network_info.py
名称: 网络信息工具
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-16
版本: V1.0.1
功能及相关说明:
  检测本机所有可用的内网 IP，过滤掉虚拟网卡和回环地址。
  供主窗口顶部 IP 显示组件调用。

改进内容:
  V1.0.1 - 初始版本

调试信息:
  已知问题: VPN 连接时可能返回 VPN 地址，需用户手动选择
'''

import socket
import ipaddress

def get_lan_ips() -> list[str]:
    """获取本机所有内网 IPv4 地址。

    Returns:
        list[str]: 内网 IP 列表，按"通用网段优先"排序
                   192.168.x.x > 10.x.x.x > 172.16-31.x.x
    """
    hostname = socket.gethostname()
    raw_ips = socket.gethostbyname_ex(hostname)[2]

    lan_ips = []
    for ip in raw_ips:
        try:
            addr = ipaddress.IPv4Address(ip)
            # 仅保留私有网段，排除回环
            if addr.is_private and not addr.is_loopback:
                lan_ips.append(ip)
        except ValueError:
            continue

    # 排序：192.168 优先（家庭/工作室最常见）
    def priority(ip: str) -> int:
        if ip.startswith("192.168."):
            return 0
        if ip.startswith("10."):
            return 1
        return 2

    lan_ips.sort(key=priority)
    return lan_ips
```

### 3.2 WebSocket 服务端

```python
r'''
文件位置: core/team/ws_server.py
名称: WebSocket 服务端
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-16
版本: V1.0.1
功能及相关说明:
  PC 中控的 WebSocket 服务端，监听 8889 端口接受安卓脚本连接。
  使用 PySide 6 的 QWebSocketServer，与主线程同事件循环。

改进内容:
  V1.0.1 - 初始版本

调试信息:
  已知问题: 无
  调试开关: logger.setLevel(logging.DEBUG)
'''

from PySide6.QtCore import QObject, Signal, QUrl
from PySide6.QtNetwork import QHostAddress
from PySide6.QtWebSockets import QWebSocketServer, QWebSocket
import json


class WSServer(QObject):
    """WebSocket 服务端，管理所有安卓设备的连接。"""

    # 信号
    device_connected = Signal(str, dict)      # device_id, device_info
    device_disconnected = Signal(str)         # device_id
    device_message = Signal(str, dict)        # device_id, message
    server_error = Signal(str)

    def __init__(self, port: int = 8889, parent=None):
        super().__init__(parent)
        self.port = port
        self.server = QWebSocketServer(
            "Hive-GreatSage PC Control",
            QWebSocketServer.SslMode.NonSecureMode,
            self
        )
        # device_id → QWebSocket
        self._connections: dict[str, QWebSocket] = {}
        # QWebSocket → device_id（反向查找）
        self._socket_to_id: dict[QWebSocket, str] = {}

        self.server.newConnection.connect(self._on_new_connection)

    def start(self) -> bool:
        """启动监听，端口冲突时自动尝试 +1。"""
        if self.server.listen(QHostAddress.SpecialAddress.Any, self.port):
            return True
        # 尝试备用端口
        if self.server.listen(QHostAddress.SpecialAddress.Any, self.port + 1):
            self.port += 1
            return True
        self.server_error.emit(
            f"端口 {self.port} 和 {self.port + 1} 都被占用，请检查"
        )
        return False

    def _on_new_connection(self):
        socket = self.server.nextPendingConnection()
        socket.textMessageReceived.connect(
            lambda msg: self._on_message(socket, msg)
        )
        socket.disconnected.connect(lambda: self._on_disconnected(socket))
        # 注意：此时还未认证，等待客户端发认证消息

    def _on_message(self, socket: QWebSocket, message: str):
        try:
            data = json.loads(message)
        except json.JSONDecodeError:
            return

        msg_type = data.get("type")

        if msg_type == "auth":
            # 安卓客户端认证
            device_id = data.get("device_id")
            token = data.get("token")
            # TODO: 调用云端 API 校验 token
            if not self._verify_token(token, device_id):
                socket.sendTextMessage(json.dumps({
                    "type": "auth_failed",
                    "reason": "invalid_token"
                }))
                socket.close()
                return

            self._connections[device_id] = socket
            self._socket_to_id[socket] = device_id
            socket.sendTextMessage(json.dumps({"type": "auth_ok"}))
            self.device_connected.emit(device_id, data.get("info", {}))
        else:
            # 业务消息
            device_id = self._socket_to_id.get(socket)
            if device_id:
                self.device_message.emit(device_id, data)

    def _on_disconnected(self, socket: QWebSocket):
        device_id = self._socket_to_id.pop(socket, None)
        if device_id:
            self._connections.pop(device_id, None)
            self.device_disconnected.emit(device_id)
        socket.deleteLater()

    def send_to_device(self, device_id: str, message: dict) -> bool:
        socket = self._connections.get(device_id)
        if socket and socket.isValid():
            socket.sendTextMessage(json.dumps(message))
            return True
        return False

    def broadcast(self, message: dict):
        """向所有已连接设备广播。"""
        text = json.dumps(message)
        for socket in self._connections.values():
            if socket.isValid():
                socket.sendTextMessage(text)

    def _verify_token(self, token: str, device_id: str) -> bool:
        """token 校验 — 实际实现调用网络验证系统 API。"""
        # TODO: 实际实现
        return bool(token)

    @property
    def connected_count(self) -> int:
        return len(self._connections)
```

### 3.3 主窗口 IP 显示组件

```python
r'''
文件位置: ui/widgets/lan_status_widget.py
名称: 局域网状态显示组件
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-16
版本: V1.0.1
功能及相关说明:
  在主窗口顶部显示本机内网 IP、端口、监听状态、已连接设备数。
  支持一键复制 IP:端口 到剪贴板。

改进内容:
  V1.0.1 - 初始版本

调试信息:
  已知问题: 无
'''

from PySide6.QtCore import Slot
from PySide6.QtGui import QGuiApplication
from PySide6.QtWidgets import (
    QHBoxLayout, QLabel, QPushButton, QComboBox, QWidget
)
from core.team.network_info import get_lan_ips


class LanStatusWidget(QWidget):
    """局域网状态条 — 显示在主窗口顶部。"""

    def __init__(self, ws_server, parent=None):
        super().__init__(parent)
        self.ws_server = ws_server

        layout = QHBoxLayout(self)
        layout.setContentsMargins(8, 4, 8, 4)

        layout.addWidget(QLabel("内网 IP:"))

        self.ip_combo = QComboBox()
        ips = get_lan_ips()
        if ips:
            self.ip_combo.addItems(ips)
        else:
            self.ip_combo.addItem("未检测到内网 IP")
        layout.addWidget(self.ip_combo)

        self.port_label = QLabel(f": {ws_server.port}")
        layout.addWidget(self.port_label)

        self.copy_btn = QPushButton("复制")
        self.copy_btn.clicked.connect(self._on_copy)
        layout.addWidget(self.copy_btn)

        layout.addStretch()

        self.status_label = QLabel("● 启动中...")
        layout.addWidget(self.status_label)

        self.device_count_label = QLabel("已连接: 0")
        layout.addWidget(self.device_count_label)

        # 连接 ws_server 信号
        ws_server.device_connected.connect(self._update_count)
        ws_server.device_disconnected.connect(self._update_count)

    @Slot()
    def _on_copy(self):
        addr = f"{self.ip_combo.currentText()}:{self.ws_server.port}"
        QGuiApplication.clipboard().setText(addr)
        self.status_label.setText(f"已复制: {addr}")

    @Slot()
    def _update_count(self):
        count = self.ws_server.connected_count
        self.device_count_label.setText(f"已连接: {count}")

    def set_listening(self):
        self.status_label.setText("● 监听中")
        self.status_label.setStyleSheet("color: green")
```

---

## 四、安卓脚本实现要点（懒人精灵 Lua）

### 4.1 配置存储

```lua
--[[
文件位置: core/team_config.lua
名称: 组队连接配置
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-16
版本: V1.0.1
功能及相关说明:
  管理 PC 中控的连接配置（IP + 端口）。
  首次启动时弹出 UI 让用户填写，之后存到本地，每次启动自动读取。

改进内容:
  V1.0.1 - 初始版本

调试信息:
  已知问题: 无
--]]

local TeamConfig = {}

local CONFIG_PATH = "/sdcard/HiveGreatSage/team_config.json"

-- 默认配置
local DEFAULT_CONFIG = {
    pc_ip = "",
    pc_port = 8889,
    auto_reconnect = true,
    reconnect_interval = 3000,  -- 3 秒
}

function TeamConfig.load()
    local file = io.open(CONFIG_PATH, "r")
    if not file then
        return DEFAULT_CONFIG
    end
    local content = file:read("*a")
    file:close()
    local ok, config = pcall(cjson.decode, content)
    if not ok then
        return DEFAULT_CONFIG
    end
    return config
end

function TeamConfig.save(config)
    -- 确保目录存在
    os.execute("mkdir -p /sdcard/HiveGreatSage")
    local file = io.open(CONFIG_PATH, "w")
    if not file then
        return false
    end
    file:write(cjson.encode(config))
    file:close()
    return true
end

function TeamConfig.is_configured(config)
    return config.pc_ip ~= nil and config.pc_ip ~= ""
end

return TeamConfig
```

### 4.2 WebSocket 连接管理

```lua
--[[
文件位置: core/comm_lan.lua
名称: 局域网通信模块
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-16
版本: V1.0.1
功能及相关说明:
  通过 WebSocket 主动连接 PC 中控，建立双向通信。
  断线后自动重连，连接成功后定时上报状态。

改进内容:
  V1.0.1 - 初始版本

调试信息:
  已知问题: 无
  调试开关: DEBUG_MODE = true
--]]

local TeamConfig = require("core.team_config")

local CommLan = {}

local DEBUG_MODE = false

local ws_handle = nil
local config = nil
local device_id = nil
local user_token = nil
local on_command_callback = nil  -- 收到 PC 指令时的回调

local function log(msg)
    if DEBUG_MODE then
        print("[CommLan] " .. msg)
    end
end

local function on_opened(handle)
    ws_handle = handle
    log("已连接 PC")
    -- 立即发送认证
    local auth_msg = {
        type = "auth",
        device_id = device_id,
        token = user_token,
        info = {
            device_name = getDeviceName(),  -- 懒人精灵自带
            screen_size = string.format("%dx%d", getScreenWidth(), getScreenHeight()),
        }
    }
    sendWebSocket(handle, cjson.encode(auth_msg))
end

local function reconnect()
    ws_handle = nil
    log("断开连接，" .. (config.reconnect_interval / 1000) .. " 秒后重连")
    setTimer(function()
        CommLan.connect()
    end, config.reconnect_interval)
end

local function on_closed(handle)
    if config.auto_reconnect then
        reconnect()
    end
end

local function on_error(handle)
    if config.auto_reconnect then
        reconnect()
    end
end

local function on_recv(handle, message)
    local ok, data = pcall(cjson.decode, message)
    if not ok then
        log("解析消息失败: " .. tostring(message))
        return
    end

    if data.type == "auth_ok" then
        log("认证成功")
    elseif data.type == "auth_failed" then
        log("认证失败: " .. (data.reason or "unknown"))
        config.auto_reconnect = false  -- 停止重连
    else
        -- 业务指令交给回调处理
        if on_command_callback then
            on_command_callback(data)
        end
    end
end

function CommLan.init(dev_id, token, callback)
    device_id = dev_id
    user_token = token
    on_command_callback = callback
    config = TeamConfig.load()
end

function CommLan.connect()
    if not TeamConfig.is_configured(config) then
        log("PC IP 未配置，跳过局域网连接")
        return false
    end
    local url = string.format("ws://%s:%d", config.pc_ip, config.pc_port)
    log("尝试连接: " .. url)
    startWebSocket(url, on_opened, on_closed, on_error, on_recv)
    return true
end

function CommLan.send(message_table)
    if not ws_handle then
        return false
    end
    sendWebSocket(ws_handle, cjson.encode(message_table))
    return true
end

function CommLan.is_connected()
    return ws_handle ~= nil
end

function CommLan.stop()
    config.auto_reconnect = false
    ws_handle = nil
end

return CommLan
```

### 4.3 首次配置 UI

懒人精灵高级版支持脚本 UI（参考懒人精灵指南），在脚本启动界面增加 PC IP 输入框：

```lua
--[[
文件位置: ui/team_config_ui.lua
名称: 局域网组队配置界面
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-04-16
版本: V1.0.1
功能及相关说明:
  提供 PC IP 输入界面，与脚本启动界面集成。
  使用懒人精灵 UI 配置语法（具体语法以懒人精灵指南为准）。

改进内容:
  V1.0.1 - 初始版本

调试信息:
  已知问题: UI 控件名称需对照懒人精灵指南最新版本调整
--]]

-- 懒人精灵的 UI 配置示例（语法以懒人精灵高级版指南为准）
-- 示意：在脚本启动界面增加以下配置项

local ui_config = [[
{
  "title": "蜂巢·大圣 — 安卓脚本",
  "items": [
    {
      "type": "TextInput",
      "key": "pc_ip",
      "label": "PC 中控 IP（工作室内网，固定 IP）",
      "placeholder": "例如 192.168.1.100",
      "default": ""
    },
    {
      "type": "TextInput",
      "key": "pc_port",
      "label": "端口",
      "placeholder": "默认 8889",
      "default": "8889"
    },
    {
      "type": "Switch",
      "key": "enable_team",
      "label": "启用局域网组队",
      "default": false
    }
  ]
}
]]

-- 用户点击「开始」后，把 UI 中的值保存到 team_config.json
-- 之后 main.lua 启动时调用 CommLan.connect() 即可
```

---

## 五、消息协议（PC ↔ 安卓）

### 5.1 消息格式

```
所有消息都是 JSON 文本，通过 WebSocket 文本帧传输。
基本结构：{ "type": "...", ...其他字段 }
```

### 5.2 协议清单

```
═══════════════════════════════════════════════════
安卓 → PC 消息
═══════════════════════════════════════════════════

# 认证（连接建立后第一条消息）
{
  "type": "auth",
  "device_id": "android-xxx",
  "token": "用户token",
  "info": {
    "device_name": "Mi 10",
    "screen_size": "1080x2400",
    "lazy_version": "X.X"
  }
}

# 状态心跳（每 10 秒一次）
{
  "type": "heartbeat",
  "status": "running",          // running / idle / error
  "current_task": "日常任务",
  "game_account": "玩家A"
}

# 任务完成上报
{
  "type": "task_completed",
  "task_id": "...",
  "result": "success",
  "data": { ... }
}

# 错误上报
{
  "type": "error",
  "level": "warning",           // warning / error / fatal
  "message": "...",
  "screenshot": "base64..."     // 可选
}

═══════════════════════════════════════════════════
PC → 安卓 消息
═══════════════════════════════════════════════════

# 认证响应
{ "type": "auth_ok" }
{ "type": "auth_failed", "reason": "invalid_token" }

# 启动任务
{
  "type": "start_task",
  "task_name": "...",
  "params": { ... }
}

# 停止任务
{ "type": "stop_task" }

# 组队指令
{
  "type": "team_command",
  "action": "follow",           // follow / lead / disband
  "leader_id": "...",
  "target_map": "..."
}

# 参数更新
{
  "type": "params_update",
  "params": { ... }
}
```

---

## 六、扩展路线图（B / D 方案后续叠加）

```
Phase 1（当前 — MVP）：方案 A
  ✓ PC 监听 WebSocket
  ✓ 安卓主动连接
  ✓ 用户手动填 IP
  ✓ PC 中控显示内网 IP

Phase 2 — 方案 D（云端中转辅助）
  当 PC 和安卓不在同一局域网时：
  · PC 中控向云端注册自己的"在线状态 + token"
  · 安卓脚本启动时先查云端 → 拿到 PC 的临时连接信息
  · 仍然走 WebSocket，但 PC 需要做内网穿透或公网部署
  · 代价：需要云端维护额外的会话表

Phase 3 — 方案 B 的变体（局域网自动发现）
  懒人精灵无法监听端口，但可以做反向扫描：
  · 安卓启动时主动扫描局域网网段
    （192.168.1.1 ~ 192.168.1.254 各试一次 ws://x:8889）
  · 找到响应正确握手的 IP → 自动写入配置
  · 代价：扫描耗时 30~60 秒，不适合频繁启动场景
  · 适用：搬到新工作室时一次性自动配置
```

---

## 七、实施清单

```
PC 中控端:
  □ 实现 core/team/network_info.py（内网 IP 检测）
  □ 实现 core/team/ws_server.py（WebSocket 服务端）
  □ 实现 ui/widgets/lan_status_widget.py（IP 显示组件）
  □ 在 ui/main_window.py 顶部加入 lan_status_widget
  □ 在 core/app.py 启动流程中初始化 ws_server
  □ 处理端口冲突 → 自动 fallback 到 8890

安卓脚本端:
  □ 实现 core/team_config.lua（配置读写）
  □ 实现 core/comm_lan.lua（WebSocket 客户端）
  □ 在脚本启动 UI 中加入 PC IP / 端口输入框
  □ 在 main.lua 启动流程中调用 CommLan.connect()
  □ 测试：断线重连、认证失败处理、长时间运行稳定性

测试场景:
  □ 工作室固定 IP 场景：填一次 IP，多次启动稳定连接
  □ 端口冲突场景：PC 8889 被占用 → 自动用 8890
  □ 安卓断网场景：网络断开 30 秒后恢复，自动重连
  □ PC 重启场景：PC 关闭再启动，安卓自动重连
  □ 大规模场景：100 台安卓同时连接 PC，无丢包
```

---

## 八、AI 协作风险标注

```
⚠ 我对懒人精灵 UI 配置语法的细节不确定
  上面 ui/team_config_ui.lua 中的 JSON 结构是我基于通用脚本框架的假设。
  懒人精灵高级版的实际 UI 配置语法可能不同（XML？特定 DSL？）。
  → 需要你查阅懒人精灵指南后修正

⚠ 我对懒人精灵能否调用 cjson 库不确定
  懒人精灵函数文档中没看到明确的 JSON 编解码函数。
  startWebSocket 的示例使用 string.format 拼接消息。
  → 需要确认是否有内置 JSON 库，或需要引入第三方库

⚠ getDeviceName / getScreenWidth 等函数名是我的假设
  懒人精灵文档应该有具体的设备信息获取函数，需要对照实际 API 调整

⚠ PySide 6 的 QWebSocketServer 我没有实测
  这是 Qt 官方的 WebSocket 模块，逻辑上正确，
  但具体 Signal 名称和参数顺序可能和我写的有偏差，需要在 IDE 中验证

实施时请以官方文档和实测结果为准，发现不一致请记录到决策日志。
```
