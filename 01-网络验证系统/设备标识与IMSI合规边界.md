---
文件位置: 01-网络验证系统/设备标识与IMSI合规边界.md
名称: 设备标识与IMSI合规边界
作者: 蜂巢·大圣 (Hive-GreatSage)
时间: 2026-05-10
版本: V1.1.0
状态: 定稿
关联文档:
  - "[[01-网络验证系统/接入契约]]"
  - "[[03-安卓脚本框架/Verify接口调用清单]]"
变更记录:
  - V1.1.0 (2026-05-10): 审查后同步——IMSI 已实现 Fernet 加密 + HMAC 哈希反查；API 不再返回原文
  - V1.0.0: Obsidian 去漂移重构生成
---

# 设备标识与IMSI合规边界

> 最后更新: 2026-05-10（审查 + 修复同步）

## 当前结论

✅ IMSI 合规方案已落地（2026-05-09）。

| 层面 | 实现 |
|------|------|
| 数据库存储 | Fernet 加密密文（`device_binding.imsi`，256字符）+ HMAC-SHA256 哈希（`device_binding.imsi_hash`，64字符） |
| API 响应 | `imsi_masked`（脱敏展示，如 `46001***1234`）+ `imsi_hash`（HMAC），不返回原文 |
| 反查能力 | 通过 `imsi_hash` 做精确匹配查询，定位后通过 `app.core.crypto.decrypt_field()` 解密 |
| 加密密钥 | 从 `SECRET_KEY` 经 HKDF 派生 Fernet key |
| 日志/审计 | `audit_service` 自动将 `imsi` 标记为敏感词并替换为 `<redacted>` |

## 源码状态（2026-05-10）

- `POST /api/device/imsi` — `device_service.upload_imsi()` 加密后写入 `binding.imsi`，同时写入 `binding.imsi_hash`
- `GET /admin/api/devices/` — `device_admin._device_response()` 解密后脱敏返回 `imsi_masked` + `imsi_hash`
- `DeviceList.vue` — 不展示 IMSI 原文，仅展示脱敏值
- 迁移 0022 — 回填加密历史明文 IMSI

## 建议方案（已实现对照）

| 建议 | 状态 |
|------|------|
| 保存 hash + last4，不保存完整明文 | ✅ `imsi_hash` + Fernet 加密 `imsi` |
| 管理员可见脱敏值，代理不可见 | ✅ `imsi_masked`，设备列表前端脱敏展示 |
| 日志禁止打印完整 IMSI | ✅ `audit_service._sanitize_metadata` 自动识别 |
| AndroidScript 显式配置 | 🔲 待 AndroidScript 开发时接入 `enable_imsi_upload` 配置 |


## 加密方案详解 (app/core/crypto.py)

### 密钥派生
SECRET_KEY (环境变量) -> HKDF(SHA256, salt, info, len=32) -> 32字节原始密钥 -> base64编码 -> Fernet key

### API
- encrypt_field(plaintext) -> Fernet token (base64, ~150字符)
- decrypt_field(token) -> 原文 | None (解密失败返回 None)

### 密钥轮换影响
- SECRET_KEY 变更 -> 新 Fernet key 无法解密旧密文
- 迁移 0022 在 SECRET_KEY 不变的前提下回填历史明文 IMSI
- 生产环境 SECRET_KEY 轮换前需先执行密文重加密

### 使用场景
- device_binding.imsi: encrypt_field(plain_imsi) 存储, imsi_hash 反查定位后 decrypt_field 解密
- 审计日志 metadata: audit_service 自动脱敏为 <redacted>, 不反查
