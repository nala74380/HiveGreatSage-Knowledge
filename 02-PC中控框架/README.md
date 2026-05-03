---
文件位置: 02-PC中控框架/README.md
名称: PC中控框架 子目录说明
作者: 蜂巢·大圣 (HiveGreatSage)
时间: 2026-04-16
版本: V1.0.2
状态: 草稿
关联文档:
  - "[[项目总大纲]]"
  - "[[待决策清单]]"
  - "[[02-PC中控框架/Verify接口调用清单]]"
  - "[[01-网络验证系统/API路由清单]]"
  - "[[01-网络验证系统/热更新端到端测试清单]]"
  - "[[03-安卓脚本框架/README]]"
变更记录:
  - V1.0.2: 同步 Verify 热更新链路小修复；补充 PCControl Verify 调用清单与热更新联动入口
  - V1.0.1: 初版 — 目录占位
---

← 返回 [[项目总大纲]]

# PC 中控框架 子目录说明

本目录存放 [[项目总大纲#三、子系统2：PC 中控框架]] 的详细设计文档。

## 当前文档

```text
02-PC中控框架/
├── README.md                         ← 本文件
├── Verify接口调用清单.md              ← 已创建，记录 PCControl 调用 Verify 的接口契约
├── 架构设计.md                        ← 已创建，待继续补齐源码级边界
├── PC安卓局域网通信方案.md             ← 已创建
├── 跨账号智能调度分析.md               ← 已创建
├── 通信协议.md                        ← 待从 PC安卓局域网通信方案 中抽出
├── UI设计/                            ← 待创建子目录
└── 打包与发布.md                      ← 待创建
```

## 当前已确认

- PCControl 已封装 Verify 登录、Token 刷新、设备列表、设备数据、参数同步、热更新检查和热更新下载接口。
- PCControl 热更新检查代码当前读取 Verify `UpdateCheckResponse.current_version` 作为服务端版本。
- Verify 源码层已将客户端热更新 `check` / `download` 查询统一到主库 `VersionRecord`。
- PCControl 热更新下载模块已有 SHA-256 校验设计入口。

## 当前待验证

- PCControl 是否能在真实 Verify 环境下登录成功。
- PC 登录是否确认不占用 Android 授权设备数。
- PCControl 是否能真实下载 PC 更新包。
- PCControl 打包为 exe 后，自更新覆盖、重启、回退是否可用。
- PCControl 参数编辑 UI 是否完整接入 `/api/params/get` 与 `/api/params/set`。
- 开发模式 mock fallback 是否可能掩盖真实 Verify 接口失败。

## 相关决策

- D003 — 核心架构原则：每游戏独立中控。
- D011 — 独立仓库模式。
- D012 — PC↔安卓局域网通信：手动 IP + WebSocket。
- D014 — 热更新版本记录迁移到主库 `VersionRecord`。

## 待推演决策

参见 [[待决策清单]] T007–T011。

当前最重要的 PC 侧待决策：

- **T007**：PC 端打包方式，PyInstaller 还是 Nuitka。
- **T008**：PC 自更新后如何安全重启、覆盖和回退。
- **PC 登录设备名额**：PC 中控是否占用 Android 授权设备数。

## 子系统关联

PC 中控依赖：
- ← [[01-网络验证系统/README|网络验证系统]]（云端中枢）

PC 中控通信：
- ←→ [[03-安卓脚本框架/README|安卓脚本]]（局域网组队）

PC 中控热更新：
- ← [[01-网络验证系统/热更新端到端测试清单]]（三端热更新联调入口）
- ← [[02-PC中控框架/Verify接口调用清单]]（PC 端 Verify 调用契约）
