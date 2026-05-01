---
文件位置: 02-PC中控框架/ADB管理模块.md
名称: ADB管理模块
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-05-01
版本: V1.0.0
状态: 草稿
关联文档:
  - "[[00-项目总控/项目总大纲]]"
变更记录:
  - V1.0.0: Obsidian 去漂移重构生成
---

# ADB 管理模块

> **文档性质：** 模块设计文档（技术定稿） **文档状态：** 定稿 v1 **最后更新：** 2026-04-19 **存放位置：** Obsidian → 02-PC中控框架/ADB管理模块.md **前置文档：** [[02-PC中控框架/架构设计|架构设计]] v2 **代码位置：** `core/utils/adb_manager.py` **关联 Claude 对话：** 02-PC中控-UI设计-0419

---

## 一、模块职责

|职责|说明|
|---|---|
|ADB 工具包管理|定位并调用 `tools/adb/platform-tools/adb.exe`|
|USB 设备连接|枚举、监测 USB 连接的安卓设备|
|TCP/IP 设备连接|切换设备到 TCP/IP 模式，通过 WiFi 连接|
|Shell 命令执行|向目标设备发送任意 shell 命令（单条/批量）|
|设备激活|部署并启动代理守护进程（ROOT 模式）|
|文件传输|push / pull 文件|
|设备信息读取|序列号、型号、Android 版本、IP 地址|

---

## 二、关键决策（已定稿）

### 决策1：ADB 工具包来源

**结论：** Google 官方 platform-tools，手动下载后放置到项目目录。

**下载地址：** https://developer.android.com/tools/releases/platform-tools

**放置路径：**

```text
HiveGreatSage-PCControl/
└── tools/
    └── adb/
        ├── platform-tools/
        │   ├── adb.exe
        │   ├── AdbWinApi.dll
        │   ├── AdbWinUsbApi.dll
        │   └── fastboot.exe
        └── README.md          ← 记录版本号和更新时间
```

**版本管理方式：** 不提交到 Git（体积大、二进制文件），通过 `README.md` 记录版本号，发版时随安装包一同分发。`.gitignore` 添加 `tools/adb/platform-tools/`。

**⚠️ 存疑（需实测）：** PyInstaller 打包时 `tools/adb/` 的相对路径处理，需验证 `_MEIPASS` 临时目录下路径是否正确。

---

### 决策2：调用方式

**结论：** 方案A — 直接 `subprocess` 调用，无第三方 Python 依赖。

**理由：**

- 无额外依赖，打包更简单
- `subprocess.run()` 足以满足所有 ADB 操作需求
- `pure-python-adb` 等库功能有限且维护不稳定

**路径定位逻辑：**

```python
_PROJ_ROOT = Path(__file__).resolve().parents[2]
ADB_EXE    = _PROJ_ROOT / "tools" / "adb" / "platform-tools" / "adb.exe"
```

以 `adb_manager.py` 文件自身位置向上两级定位项目根，不依赖工作目录，打包后同样有效。

---

### 决策3：连接模式

**结论：** USB 和 TCP/IP 双模式均支持，上层传入不同格式的 `serial` 即可区分。

|模式|Serial 格式|示例|
|---|---|---|
|USB|硬件序列号|`SN4F2A8C1E`|
|TCP/IP|`ip:port`|`192.168.1.101:5555`|

**典型 TCP/IP 使用流程：**

```text
1. USB 连接设备
2. 调用 enable_tcpip(serial)     → 设备开始监听 5555 端口
3. 调用 get_ip_address(serial)   → 获取设备 WiFi IP
4. 拔掉 USB
5. 调用 connect_tcpip(ip)        → 通过 WiFi 连接
6. 后续所有操作使用 ip:port 作为 serial
```

---

## 三、激活命令序列（已定稿）

激活操作 = 向设备部署并启动代理守护进程，依次执行以下三条命令：

```bash
# 步骤1：复制代理程序到可执行目录
adb shell cp /sdcard/Download/proxy /data/local/tmp/proxy

# 步骤2：赋予执行权限
adb shell chmod 777 /data/local/tmp/proxy

# 步骤3：以守护进程方式启动
adb shell /data/local/tmp/proxy --daemon
```

**失败处理：** 任一步骤失败立即中止，将失败步骤序号、命令内容和 stderr 输出返回给 UI 层，UI 层在弹窗中展示给用户。

**前置条件：**

- 设备处于 `device` 状态（已授权）
- `/sdcard/Download/proxy` 文件已存在于设备（需提前推送或用户手动放置）

**⚠️ 存疑（需实测）：**

- `--daemon` 参数是否在设备重启后自动恢复？如不能，需研究开机自启方案
- 部分厂商 ROM 的 `/data/local/tmp/` 权限限制

---

## 四、类结构（core/utils/adb_manager.py）

```text
AdbManager
  │
  ├── 初始化
  │     └── __init__(adb_path=None)      检测 adb.exe 是否存在
  │
  ├── Server 管理
  │     ├── start_server() → bool
  │     └── kill_server()  → bool
  │
  ├── 设备枚举
  │     ├── list_devices() → list[DeviceInfo]
  │     └── get_device(serial) → DeviceInfo | None
  │
  ├── TCP/IP 连接
  │     ├── enable_tcpip(serial, port=5555) → bool
  │     ├── connect_tcpip(ip, port=5555)    → bool
  │     ├── disconnect_tcpip(ip, port=5555) → bool
  │     └── disconnect_all()                → bool
  │
  ├── Shell 命令
  │     ├── shell(serial, command, timeout) → AdbResult
  │     └── shell_batch(serial, commands, stop_on_error) → list[AdbResult]
  │
  ├── 文件传输
  │     ├── push(serial, local, remote) → bool
  │     └── pull(serial, remote, local) → bool
  │
  ├── 激活操作
  │     ├── activate_device(serial) → (bool, str)
  │     └── batch_activate(serials) → dict[str, (bool, str)]
  │
  └── 设备信息读取
        ├── get_serial_no(serial)       → str
        ├── get_model(serial)           → str
        ├── get_android_version(serial) → str
        └── get_ip_address(serial)      → str

数据类
  ├── AdbResult       (success, returncode, stdout, stderr)
  ├── DeviceInfo      (serial, mode, state, model, product, extra)
  └── ConnMode(Enum)  USB | TCPIP
```

---

## 五、与 UI 层的集成点

### 5.1 初始化（`core/app.py`）

```python
self.adb = AdbManager()
self.adb.start_server()   # 程序启动时调用一次
```

### 5.2 ADB 操作必须在工作线程执行

`subprocess` 调用会阻塞，不能在 Qt 主线程执行，否则 UI 卡死。

```python
# ui/widgets/device_table_widget.py 示意
class ActivateWorker(QThread):
    finished = Signal(str, bool, str)   # serial, success, message

    def __init__(self, adb, serial):
        super().__init__()
        self.adb, self.serial = adb, serial

    def run(self):
        ok, msg = self.adb.activate_device(self.serial)
        self.finished.emit(self.serial, ok, msg)

def on_activate_clicked(self, serial):
    worker = ActivateWorker(self.app.adb, serial)
    worker.finished.connect(self.on_activate_done)
    worker.start()

def on_activate_done(self, serial, ok, msg):
    # 回到主线程，安全更新 UI
    if ok:
        QMessageBox.information(self, "激活成功", f"{serial}\n{msg}")
    else:
        QMessageBox.warning(self, "激活失败", f"{serial}\n{msg}")
    self.refresh_activated_column(serial)
```

### 5.3 TCP/IP 连接流程（右键菜单触发）

```python
def on_switch_to_tcpip(self, serial):
    ip = self.app.adb.get_ip_address(serial)
    if not ip:
        QMessageBox.warning(self, "错误", "无法获取 WiFi IP，请确认设备已连接 WiFi")
        return
    self.app.adb.enable_tcpip(serial)
    QMessageBox.information(
        self, "TCP/IP 模式",
        f"已切换\n设备 IP：{ip}:5555\n\n可拔掉 USB，使用右键菜单「TCP 连接」重新连入"
    )
```

---

## 六、tools/adb/README.md 模板

```markdown
# ADB Platform Tools

来源：https://developer.android.com/tools/releases/platform-tools
平台：Windows
当前版本：35.0.2（2026-04-19 更新）

## 更新方式
1. 从上方链接下载最新 Windows platform-tools.zip
2. 解压，用新文件覆盖本目录下的 platform-tools/ 文件夹
3. 更新本文件中的版本号和更新日期

## 版本历史
| 版本    | 更新日期       | 更新人  |
|--------|--------------|---------|
| 35.0.2 | 2026-04-19   | Hailin  |
```

---

## 七、.gitignore 新增条目

```gitignore
# ADB 工具包二进制（通过 README 记录版本，随安装包分发）
tools/adb/platform-tools/
```

---

## 八、待确认 / 风险项

|项目|风险等级|说明|
|---|---|---|
|PyInstaller 打包后路径|🟡 中|`_MEIPASS` 临时目录路径需实测|
|守护进程重启后是否存活|🟡 中|`--daemon` 重启不一定自动恢复|
|厂商 ROM `/data/local/tmp` 权限|🟡 中|部分机型可能拒绝写入或执行|
|批量激活并发控制|🟢 低|目前顺序执行，后续可改线程池|
|TCP/IP 断线自动重连|🟢 低|心跳超时后的重连策略待设计|