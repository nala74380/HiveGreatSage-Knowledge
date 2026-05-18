# 设备编号与 ADB 连接映射融合方案

## 1. 当前任务理解

本文用于融合三类确认方式：

```text
方案 1：LAN WebSocket IP 与 TCP ADB IP 自动匹配
方案 2：USB / TCP ADB 设备由人工显式绑定到 device_id
方案 3：PC 中控通过 ADB 读取安卓端本地 device_identity.json 确认 device_id
```

融合目标不是替代 Verify 设备绑定规则，而是在 PC 中控本地建立一层清晰的 ADB 连接映射能力。

---

## 2. 已确认信息

### 2.1 Verify 设备绑定主键

已确认：

```text
Verify 设备绑定主键 = user_id + game_project_id + device_id
```

其中：

```text
device_id = 安卓端客户填写的设备编号
```

安卓端登录、心跳上传 `device_id`，后端保存并通过设备列表接口下发给 PC 中控。

### 2.2 ADB serial 来源

已确认：

```text
adb serial 只能由 PC 中控本机通过 ADB 查询获得。
```

例如：

```text
adb devices -l
R58M1234567        device product:xxx model:xxx
192.168.2.28:5555 device product:xxx model:xxx
```

因此：

```text
device_id ≠ adb serial
```

除非经过明确确认，不允许把二者当成同一个东西。

### 2.3 connection_type / connection_label 口径

已确认：

```text
connection_type / connection_label 是 PC 中控侧本地 ADB 查询后的展示信息。
```

它们不属于 Verify 绑定主键，不由安卓端登录/心跳作为绑定字段上报。

---

## 3. 非目标

本方案不做：

```text
不把 adb serial 上传 Verify 作为设备绑定主键。
不把 connection_type / connection_label 写入 Verify 绑定唯一性。
不通过硬件序列号自动生成 device_id。
不使用隐藏设备唯一标识。
不在 PC 中控主表中用 ADB 设备替代 Verify 设备列表。
不把 LAN 在线成员直接混入 Verify 设备主表。
```

---

## 4. 核心设计

新增本地映射层：

```text
AdbLinkManager
```

职责：

```text
扫描 PC 本地 ADB 设备。
读取本地已保存映射。
尝试 LAN IP 自动匹配。
尝试 ADB 读取安卓端 device_identity.json。
支持人工显式绑定。
输出 device_id -> ADB 连接展示信息。
```

输出字段：

```text
device_id
adb_serial
connection_type
connection_label
match_method
match_confidence
verified_at
source
```

---

## 5. 三方案融合后的优先级

融合后不是三选一，而是按可信度分层确认。

### 5.1 第一优先级：人工显式绑定

适用场景：

```text
USB ADB 设备。
TCP ADB IP 不稳定。
同一局域网内存在多个设备。
无法通过 ADB 读取安卓端 identity 文件。
```

绑定流程：

```text
用户在 PC 中控选择 Verify 设备 device_id。
用户在本机 ADB 设备列表中选择 adb_serial。
用户点击“绑定本机 ADB 连接”。
PC 中控保存到 config/device_adb_links.json。
```

可信度：

```text
manual = high
```

是否可自动覆盖：

```text
不允许被 LAN IP 弱匹配自动覆盖。
不允许被 device_id == adb serial 自动覆盖。
允许用户手动重新绑定。
```

---

### 5.2 第二优先级：ADB 读取安卓端 device_identity.json

适用场景：

```text
安卓端脚本框架可在本机写入公开身份文件。
PC 中控通过 adb shell cat 可读取。
```

安卓端建议写入文件：

```text
/sdcard/HiveGreatSage/device_identity.json
```

内容：

```json
{
  "device_id": "A118",
  "project_uuid": "00000000-0000-0000-0000-000000000001",
  "client": "HiveGreatSage-AndroidScript",
  "updated_at": "2026-05-18T08:50:00"
}
```

注意：

```text
该文件不得包含账号密码。
该文件不得包含授权 token。
该文件不得包含真实游戏密码。
该文件只用于 PC 中控本地确认 adb_serial 与 device_id 的关系。
```

PC 中控流程：

```text
遍历 adb devices。
对每个 adb_serial 执行 adb -s <serial> shell cat /sdcard/HiveGreatSage/device_identity.json。
解析 device_id。
如果 device_id 存在于 Verify 设备列表，则建立本地映射。
```

可信度：

```text
adb_identity = high
```

冲突处理：

```text
如果已存在 manual 绑定，不自动覆盖。
如果 identity 读出的 device_id 与已有非 manual 映射冲突，标记冲突，等待用户确认。
```

---

### 5.3 第三优先级：LAN WebSocket IP 与 TCP ADB IP 匹配

适用场景：

```text
安卓端通过 LAN WebSocket 连接 PC 中控。
ADB 设备以 tcpip 方式连接，例如 192.168.2.28:5555。
LAN WebSocket 连接 IP 与 ADB serial IP 一致。
```

匹配规则：

```text
TeamMember.device_id = A118
TeamMember.peer_ip = 192.168.2.28
ADB serial = 192.168.2.28:5555
=> A118 可匹配到 192.168.2.28:5555
```

可信度：

```text
lan_ip = medium
```

风险：

```text
IP 可能变化。
多网卡、代理、NAT、端口转发可能导致误判。
同一设备重连后需要重新校验。
```

冲突处理：

```text
不覆盖 manual。
不覆盖 adb_identity。
只填补空映射或低可信映射。
如果同一 IP 对应多个 device_id，标记冲突，不自动绑定。
```

---

### 5.4 第四优先级：device_id == adb serial 严格匹配

适用场景极少，只作为兜底：

```text
用户刚好把 device_id 填成 ADB serial。
```

可信度：

```text
strict_equal = low_to_medium
```

限制：

```text
只能作为兜底自动提示。
不应作为主要确认方式。
不覆盖 manual / adb_identity / lan_ip。
```

---

## 6. 本地映射文件

文件路径：

```text
config/device_adb_links.json
```

结构：

```json
{
  "A118": {
    "device_id": "A118",
    "adb_serial": "192.168.2.28:5555",
    "connection_type": "tcp",
    "connection_label": "192.168.2.28:5555",
    "match_method": "lan_ip",
    "match_confidence": "medium",
    "source": "pccontrol",
    "verified_at": "2026-05-18T08:50:00",
    "manual_locked": false
  },
  "A119": {
    "device_id": "A119",
    "adb_serial": "R58M1234567",
    "connection_type": "usb",
    "connection_label": "R58M1234567",
    "match_method": "manual",
    "match_confidence": "high",
    "source": "operator",
    "verified_at": "2026-05-18T08:51:00",
    "manual_locked": true
  }
}
```

---

## 7. 模块边界

### 7.1 DeviceManager

职责：

```text
从 Verify API 拉取设备列表。
合并本地设备元数据。
请求 AdbLinkManager 注入本地 ADB 连接展示。
```

不负责：

```text
不直接执行复杂 ADB 映射策略。
不直接保存 device_adb_links.json。
不猜测 adb serial 与 device_id 的关系。
```

### 7.2 AdbLinkManager

职责：

```text
扫描 ADB 设备。
读取 / 保存 device_adb_links.json。
融合 manual / adb_identity / lan_ip / strict_equal 多种确认方式。
输出 device_id 对应的本地连接信息。
```

### 7.3 TeamManager

职责：

```text
维护 LAN WebSocket 在线成员。
提供 device_id 与 peer_ip。
```

不负责：

```text
不写 Verify 绑定。
不写 ADB 映射文件。
```

### 7.4 AndroidScript

职责：

```text
登录时使用客户填写的 device_id。
心跳时上报 device_id。
可选：写入 /sdcard/HiveGreatSage/device_identity.json，供 PC 中控通过 ADB 读取。
```

不负责：

```text
不上传 adb serial 作为 Verify 绑定主键。
不上传 connection_type / connection_label 作为绑定字段。
```

### 7.5 DevicePage

职责：

```text
主表显示 Verify 设备列表。
右侧侧栏显示 LAN 在线成员和 ADB 映射状态。
展示 connection_type / connection_label。
提供人工绑定入口。
```

---

## 8. UI 设计

### 8.1 设备主表

建议列：

```text
编号
连接标识
连接类型
角色
状态
激活
当前任务
等级
战力
区服
心跳
备注
```

其中：

```text
编号 = device_id
连接标识 = AdbLinkManager 输出的 connection_label
连接类型 = AdbLinkManager 输出的 connection_type
```

### 8.2 右侧 LAN 在线成员区

展示：

```text
LAN 在线成员数量
device_id
peer_ip
是否发现同 IP TCP ADB
是否已绑定 adb_serial
匹配方式：manual / adb_identity / lan_ip / strict_equal / none
```

### 8.3 设备设置弹窗中的 ADB 绑定页

新增页签或区域：

```text
本机 ADB 连接
```

内容：

```text
当前 device_id
当前绑定 adb_serial
当前 connection_type / connection_label
可选 ADB 设备列表
按钮：绑定当前 ADB 设备
按钮：解除本机 ADB 绑定
按钮：尝试从安卓端读取 device_identity.json
```

---

## 9. 冲突处理

### 9.1 同一 device_id 对应多个 adb_serial

处理：

```text
如果存在 manual_locked=true，以 manual 为准。
否则标记 conflict，不自动选择。
UI 提示用户选择。
```

### 9.2 同一 adb_serial 对应多个 device_id

处理：

```text
标记 conflict。
不写入 device_adb_links.json。
右侧侧栏显示冲突。
```

### 9.3 LAN IP 与 adb_identity 结果不一致

处理：

```text
adb_identity 优先。
lan_ip 结果标记为弱匹配，不覆盖。
```

---

## 10. 执行阶段

### P3.4-a：文档与边界冻结

```text
确认本方案。
确认 connection_type / connection_label 只来自 PC 中控本地。
确认 device_id 是唯一绑定口径。
```

### P3.4-b：AdbLinkManager 骨架

```text
新增 core/device/adb_link_manager.py。
新增 config/device_adb_links.json 读写。
实现 manual link 数据结构。
DeviceManager 改为依赖 AdbLinkManager，不直接猜测。
```

### P3.4-c：LAN IP 自动匹配

```text
从 TeamManager.members 获取 device_id + peer_ip。
从 ADB devices 获取 tcp serial。
匹配 IP 后生成 lan_ip 映射。
```

### P3.4-d：ADB identity 文件读取

```text
安卓端写入 /sdcard/HiveGreatSage/device_identity.json。
PC 中控通过 adb shell cat 读取并解析。
生成 adb_identity 映射。
```

### P3.4-e：UI 人工绑定

```text
设备设置弹窗新增本机 ADB 连接页。
支持选择 ADB 设备并绑定到当前 device_id。
支持解除绑定。
支持显示冲突。
```

### P3.4-f：测试补齐

```text
测试 AdbLinkManager 读写。
测试 manual 优先级。
测试 adb_identity 优先级。
测试 lan_ip 匹配。
测试冲突不自动绑定。
测试 DeviceManager 注入连接展示。
```

---

## 11. 最终结论

融合后的最终口径：

```text
Verify 只管 device_id 设备绑定。
PC 中控只把 ADB 作为本地连接能力。
AdbLinkManager 负责 device_id 与 adb_serial 的本地映射。
人工绑定、ADB identity、LAN IP 三种确认方式融合使用。
任何时候都不把 adb serial、connection_label、隐藏标识当作 Verify 设备主键。
```
