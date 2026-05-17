---
文件名: UI边界重构执行方案.md
项目: 蜂巢·大圣 HiveGreatSage
模块: HiveGreatSage-PCControl
版本: V1.0.0
状态: 草案 / 待评审 / 不代表已实施
生成时间: 2026-05-18
关联文档:
  - 02-PC中控框架/UI边界重构方案.md
执行原则: 小步提交、可回退、先冻结边界再改代码、禁止把未确认内容写成已实现
---

# UI边界重构执行方案

## 0. 执行方案定位

本文是 `UI边界重构方案.md` 的落地执行方案。

`UI边界重构方案.md` 负责回答：

```text
边界是什么？
哪些能做？
哪些不能做？
全局设置、设备设置、批量设置各自管什么？
```

本文负责回答：

```text
先改什么？
后改什么？
每一步改哪些文件？
每一步如何验收？
失败如何回退？
如何避免继续边界漂移？
```

---

## 1. 总执行原则

### 1.1 已确认原则

1. 不做远程控制、投屏、scrcpy、公网远控、Relay 远控。
2. 当前账号来源只允许：
   - 手动录入游戏账号。
   - 客户外部账号数据库。
3. 全局设置只配置平台附加能力、客户环境、本地运行环境、日志、更新、ADB、接码、邮箱等。
4. 全局设置不允许放游戏账号表、游戏任务参数、物品、交易、制造、铸币等游戏运行配置。
5. 设备设置负责单设备游戏运行配置。
6. 批量设置负责多设备批量应用配置。
7. 账号设置页里的账号是游戏账号，不是 PC 登录账号，不是后端用户账号。
8. 密码默认隐藏，但必须允许显示、隐藏、复制、编辑、受控明文导出。
9. 设备管理页主操作工具栏放到底部。
10. 本地 profile 只能作为草稿或缓存，不能直接声称为最终真相源。

### 1.2 严禁事项

```text
严禁把无证据账号来源写进 UI。
严禁把未实现模块写成已完成。
严禁把游戏配置塞进全局设置。
严禁把客户外部账号数据库里的具体账号复制成全局设置的账号表。
严禁在日志、诊断包、默认导出中暴露真实密码。
严禁在本次重构中加入远控/投屏入口。
```

### 1.3 执行方式

本次重构必须采用小步拆分，不允许一次性大改：

```text
阶段 0：冻结边界，只改文档和入口约束。
阶段 1：设备页布局下移工具栏，补右侧侧栏骨架。
阶段 2：全局设置拆分，只保留平台附加能力。
阶段 3：设备设置对话框落地。
阶段 4：账号设置页密码显示/隐藏落地。
阶段 5：批量设置对话框升级。
阶段 6：客户外部账号数据库配置页落地。
阶段 7：旧混合入口清理。
阶段 8：测试、验收、回退预案固化。
```

---

## 2. 分支与提交策略

### 2.1 建议分支

建议在 `HiveGreatSage-PCControl` 仓库新建功能分支：

```text
feature/ui-boundary-refactor
```

如果要进一步拆分，可以按阶段拆：

```text
feature/ui-boundary-p0-freeze
feature/ui-boundary-p1-device-page
feature/ui-boundary-p2-global-settings
feature/ui-boundary-p3-device-settings
feature/ui-boundary-p4-batch-settings
```

### 2.2 提交粒度

建议每个阶段至少一个提交，不建议把 UI、配置模型、账号数据库、密码处理全部揉成一个提交。

推荐提交格式：

```text
docs(pccontrol): freeze UI boundary rules
refactor(ui): move device actions to bottom toolbar
feat(ui): add device side panel skeleton
refactor(settings): split global settings dialog
feat(settings): add device settings dialog skeleton
feat(account): add password reveal and mask behavior
feat(batch): add batch settings diff preview skeleton
feat(external-account): add customer database settings page skeleton
```

---

## 3. P0：冻结边界

### 3.1 目标

先阻止继续漂移。P0 不做大 UI 改造，只做文档、入口命名、旧弹窗冻结标记。

### 3.2 修改范围

建议修改或新增：

```text
HiveGreatSage-Knowledge:
  02-PC中控框架/UI边界重构方案.md
  02-PC中控框架/UI边界重构执行方案.md

HiveGreatSage-PCControl:
  docs/pc_control/UI_BOUNDARY.md              # 建议新增
  ui/widgets/settings_dialog.py               # 加冻结说明，不继续新增游戏配置
  README.md 或 docs/README.md                 # 视实际仓库结构决定是否补入口说明
```

### 3.3 执行步骤

1. 在 PCControl 仓库新增 `docs/pc_control/UI_BOUNDARY.md`。
2. 明确三类设置入口：
   - 全局设置。
   - 设备设置。
   - 批量设置。
3. 在旧 `ui/widgets/settings_dialog.py` 文件头加入说明：
   - 当前是历史混合弹窗。
   - 后续不再新增游戏配置。
   - 新全局设置迁移到 `ui/dialogs/global_settings_dialog.py`。
   - 游戏配置迁移到 `ui/dialogs/device_settings_dialog.py` 和 `ui/dialogs/batch_settings_dialog.py`。
4. 搜索仓库内是否出现不允许的账号来源词。
5. 若发现无证据账号来源词，删除或改成“待设计：平台托管游戏账号库”。

### 3.4 验收标准

```text
- [ ] PCControl 仓库有 UI_BOUNDARY.md。
- [ ] 旧 settings_dialog.py 有冻结说明。
- [ ] 文档中没有无证据账号来源。
- [ ] 全局设置禁止项明确。
- [ ] 新功能不再进入旧 settings_dialog.py。
```

### 3.5 回退方式

P0 主要是文档和注释，回退方式简单：

```text
git revert <P0_commit>
```

---

## 4. P1：设备管理页布局重构

### 4.1 目标

把设备页从“顶部工具栏 + 表格”调整为：

```text
顶部状态/筛选
中部设备表格 + 右侧侧栏
底部主操作工具栏
```

### 4.2 修改范围

当前方案建议新增：

```text
ui/pages/device_page.py
ui/widgets/device_bottom_toolbar.py
ui/widgets/device_side_panel.py
ui/widgets/device_table/device_table_view.py
ui/widgets/device_table/device_table_model.py
ui/widgets/device_table/device_table_columns.py
ui/widgets/device_table/device_context_menu.py
```

当前可能需要拆分：

```text
ui/main_window.py
```

### 4.3 执行步骤

#### 步骤 1：先抽出 DevicePage 容器

从 `ui/main_window.py` 中拆出设备页相关 UI 代码，先不改视觉，只移动代码位置。

目标：

```text
ui/pages/device_page.py
  class DevicePage(QWidget):
      负责设备页整体布局装配
```

验收：

```text
- [ ] 主窗口仍能打开设备页。
- [ ] 原有设备表格仍能展示。
- [ ] 原有刷新、筛选、右键操作不丢失。
```

#### 步骤 2：新增 DeviceBottomToolbar

新增底部工具栏组件：

```text
ui/widgets/device_bottom_toolbar.py
  class DeviceBottomToolbar(QWidget):
      全选
      反选
      清空选择
      批量设置
      启动脚本
      停止脚本
      重启脚本
      激活设备
      解绑设备
      刷新
      导出CSV
      导出诊断
```

执行要求：

```text
只移动现有动作入口，不改变业务逻辑。
不新增远控/投屏按钮。
```

验收：

```text
- [ ] 批量设置按钮在底部。
- [ ] 启动、停止、重启、激活、导出等按钮在底部。
- [ ] 顶部不再保留重复批量按钮。
- [ ] 已选设备数能在底部显示。
```

#### 步骤 3：新增 DeviceSidePanel 骨架

新增右侧侧栏：

```text
ui/widgets/device_side_panel.py
  DeviceStatsPanel
  DeviceGroupPanel
  ColumnVisibilityPanel
  RefreshControlPanel
  SelectedSummaryPanel
```

第一阶段只做骨架和静态数据，第二阶段再接真实统计。

验收：

```text
- [ ] 右侧栏能显示。
- [ ] 不影响设备表格宽度基础可用性。
- [ ] 统计为空时显示 0 或 “待同步”。
- [ ] 不把游戏组队功能放进设备分组。
```

### 4.4 测试建议

```text
python -m compileall -q .
pytest -q
```

手工测试：

```text
- 打开主窗口。
- 查看设备页是否显示。
- 搜索设备。
- 勾选设备。
- 点击底部全选/反选/清空选择。
- 点击刷新。
- 打开右键菜单。
- 确认没有远控/投屏入口。
```

### 4.5 回退方式

如果设备页拆分导致主窗口不可用：

```text
git revert <P1_commit>
```

或临时在 `main_window.py` 中恢复旧 DevicePage 入口。

---

## 5. P2：全局设置拆分

### 5.1 目标

建立真正的全局设置入口，只处理平台附加能力和客户环境配置。

### 5.2 修改范围

新增：

```text
ui/dialogs/global_settings_dialog.py
ui/dialogs/global_settings/pages/server_settings_page.py
ui/dialogs/global_settings/pages/network_settings_page.py
ui/dialogs/global_settings/pages/external_account_db_page.py
ui/dialogs/global_settings/pages/sms_provider_page.py
ui/dialogs/global_settings/pages/mail_service_page.py
ui/dialogs/global_settings/pages/adb_settings_page.py
ui/dialogs/global_settings/pages/log_settings_page.py
ui/dialogs/global_settings/pages/update_settings_page.py
ui/dialogs/global_settings/pages/local_cache_page.py
```

新增或调整：

```text
core/settings/global_settings_model.py
core/settings/settings_codec.py
core/settings/settings_validator.py
```

暂不删除：

```text
ui/widgets/settings_dialog.py
```

### 5.3 执行步骤

1. 新建 `GlobalSettingsDialog` 骨架。
2. 先迁移当前已有的 server/network/sync/adb/update/log 配置。
3. 新增客户外部账号数据库配置页，但先只做 UI 和本地配置模型。
4. 新增接码平台、邮箱服务页面骨架。
5. 主窗口“全局设置”入口改为打开新 `GlobalSettingsDialog`。
6. 旧 `settings_dialog.py` 暂时不删除，但不再作为主入口。

### 5.4 严禁事项

```text
- 不迁移游戏账号表到全局设置。
- 不迁移任务参数到全局设置。
- 不迁移物品、交易、制造、铸币到全局设置。
- 不在全局设置中编辑具体游戏账号。
```

### 5.5 验收标准

```text
- [ ] 点击全局设置打开新 GlobalSettingsDialog。
- [ ] 全局设置页面没有游戏账号表。
- [ ] 全局设置页面没有任务路线、物品、交易、制造、铸币。
- [ ] 客户外部账号数据库页只包含连接、字段映射、拉取策略、连接测试。
- [ ] 客户数据库密码不写入 default.yaml。
- [ ] 日志不打印客户数据库密码。
```

### 5.6 回退方式

如果新全局设置不可用：

```text
- 保留旧 settings_dialog.py。
- 主窗口入口临时切回旧 settings_dialog.py。
- 新文件保留但不启用。
```

---

## 6. P3：设备设置对话框

### 6.1 目标

建立单设备游戏运行配置入口。

### 6.2 修改范围

新增：

```text
ui/dialogs/device_settings_dialog.py
ui/dialogs/device_settings/pages/main_settings_page.py
ui/dialogs/device_settings/pages/account_settings_page.py
ui/dialogs/device_settings/pages/task_settings_page.py
ui/dialogs/device_settings/pages/item_settings_page.py
ui/dialogs/device_settings/pages/purchase_settings_page.py
ui/dialogs/device_settings/pages/trade_settings_page.py
ui/dialogs/device_settings/pages/craft_settings_page.py
ui/dialogs/device_settings/pages/mint_settings_page.py
ui/dialogs/device_settings/pages/misc_settings_page.py
```

新增：

```text
core/settings/device_profile_model.py
core/settings/settings_validator.py
```

### 6.3 执行步骤

1. 新建大尺寸 `DeviceSettingsDialog`。
2. 采用 QTabWidget 多页签。
3. 底部按钮区采用：
   - 重置当前页面。
   - 全部重置。
   - 保存当前页面。
   - 全部保存。
   - 关闭。
4. 先实现页面骨架，不急于接全部后端接口。
5. 设备表双击或右键“编辑/设置”打开该对话框。
6. 本地 profile 只作为草稿，字段必须包含：
   - `draft_id`
   - `device_fingerprint`
   - `synced`
   - `synced_at`
   - `remote_version` 或 `remote_revision`（若后端未提供则标记待确认）

### 6.4 保存策略

当前后端配置接口具体形态待确认，因此执行时分两层：

```text
第一层：本地草稿保存。
第二层：后端保存接口适配。
```

在接口未确认前，不得写：

```text
已完成云端配置保存
已完成三端闭环
```

只能写：

```text
本地草稿保存已完成
后端配置接口待联调
```

### 6.5 验收标准

```text
- [ ] 单设备双击打开设备设置。
- [ ] 设备设置不是全局设置。
- [ ] 有主要设置、账号设置、任务设置等页签。
- [ ] 保存时能生成本地草稿。
- [ ] 本地草稿不被称为最终真相源。
- [ ] 后端接口未接通时 UI 明确提示“待联调”。
```

---

## 7. P4：账号设置页与密码行为

### 7.1 目标

实现游戏账号设置页，并满足真实密码可显示/隐藏/复制/编辑的需求。

### 7.2 修改范围

重点文件：

```text
ui/dialogs/device_settings/pages/account_settings_page.py
ui/widgets/account_table/password_delegate.py       # 建议新增
ui/widgets/account_table/account_table_model.py     # 建议新增
core/settings/device_profile_model.py
```

### 7.3 字段范围

当前只允许：

```text
账号来源：手动输入 / 客户外部账号数据库
```

账号表建议字段：

```text
序号
启用
账号来源
账号类型
游戏账号
游戏密码
验证邮箱
邮箱密码
大区
小区
主角色
职业
账号状态
备注
```

### 7.4 密码行为

必须实现：

```text
- 默认显示 ******。
- 右键单行显示真实密码。
- 右键单行隐藏真实密码。
- 右键复制真实密码。
- 顶部按钮显示全部密码。
- 顶部按钮隐藏全部密码。
- 双击编辑真实密码。
- 编辑完成后自动恢复掩码。
```

### 7.5 导出行为

必须实现两个导出模式：

```text
脱敏导出：默认。
明文导出：必须二次确认。
```

明文导出确认项：

```text
[ ] 我确认当前环境可信
[ ] 我确认该文件不会上传到公开渠道
[ ] 我确认需要明文密码用于客户交付或人工核对
```

### 7.6 禁止事项

```text
- 不把真实密码写入 default.yaml。
- 不在日志打印真实密码。
- 不在诊断包默认包含真实密码。
- 不把客户外部数据库全部账号复制到全局设置。
- 不出现无证据账号来源选项。
```

### 7.7 验收标准

```text
- [ ] 密码默认隐藏。
- [ ] 单行密码可显示。
- [ ] 单行密码可隐藏。
- [ ] 单行密码可复制。
- [ ] 全部密码可显示。
- [ ] 全部密码可隐藏。
- [ ] 编辑密码后自动恢复掩码。
- [ ] 导出默认脱敏。
- [ ] 明文导出有二次确认。
- [ ] 没有无证据账号来源。
```

---

## 8. P5：批量设置对话框升级

### 8.1 目标

把当前早期批量弹窗升级为真正的多设备字段级批量设置入口。

### 8.2 修改范围

新增或重构：

```text
ui/dialogs/batch_settings_dialog.py
ui/dialogs/batch_settings/pages/batch_account_page.py
ui/dialogs/batch_settings/pages/batch_task_page.py
ui/dialogs/batch_settings/pages/batch_param_page.py
ui/dialogs/batch_settings/pages/diff_preview_page.py
ui/dialogs/batch_settings/pages/apply_result_page.py
core/settings/batch_profile_model.py
```

旧文件：

```text
ui/widgets/batch_dialog.py
```

处理方式：

```text
先保留，逐步迁移。
新入口稳定后，旧入口废弃或变成兼容入口。
```

### 8.3 执行步骤

1. 新建 `BatchSettingsDialog`。
2. 增加目标设备摘要。
3. 增加字段级勾选。
4. 增加账号批量页：
   - 保持不变。
   - 手动批量导入。
   - 从客户外部数据库分配。
5. 增加参数批量页。
6. 增加差异预览页。
7. 增加应用结果页。
8. 将底部工具栏的“批量设置”按钮指向新对话框。

### 8.4 批量设置硬规则

```text
- 未勾选字段不得覆盖。
- 密码字段默认不批量覆盖。
- 密码字段批量覆盖必须二次确认。
- 每台设备必须有应用结果。
- 后端接口未确认前，先做本地差异预览和草稿。
```

### 8.5 验收标准

```text
- [ ] 可显示目标设备摘要。
- [ ] 字段级勾选可用。
- [ ] 未勾选字段保持原值。
- [ ] 支持手动批量导入账号。
- [ ] 支持从客户外部数据库分配账号。
- [ ] 密码批量覆盖有二次确认。
- [ ] 有差异预览。
- [ ] 有每台设备应用结果。
```

---

## 9. P6：客户外部账号数据库配置

### 9.1 目标

让客户能在全局设置中配置自己的账号数据库连接，但不在全局设置中编辑具体游戏账号。

### 9.2 修改范围

新增：

```text
core/external_account_db/external_account_config.py
core/external_account_db/external_account_connector.py
core/external_account_db/external_account_field_mapping.py
core/external_account_db/external_account_service.py
core/external_account_db/external_account_errors.py
ui/dialogs/global_settings/pages/external_account_db_page.py
```

### 9.3 执行步骤

1. 实现配置模型。
2. 实现字段映射模型。
3. 实现连接测试接口。
4. 实现拉取样例账号摘要。
5. 设备设置账号页接入“客户外部账号数据库”来源。
6. 批量设置账号页接入“从客户外部数据库分配”。

### 9.4 支持范围分级

第一阶段：

```text
HTTP API 模式或 mock connector，先验证 UI 与字段映射。
```

第二阶段：

```text
MySQL / PostgreSQL 连接。
```

第三阶段：

```text
按客户项目适配复杂字段映射。
```

### 9.5 验收标准

```text
- [ ] 全局设置可保存客户数据库连接配置。
- [ ] 可测试连接。
- [ ] 可配置字段映射。
- [ ] 可拉取一条样例账号摘要。
- [ ] 不在全局设置中展示完整账号表。
- [ ] 不在日志输出数据库密码或游戏密码。
```

---

## 10. P7：清理旧混合入口

### 10.1 目标

删除或废弃旧的混合设置入口，避免用户和开发者继续误用。

### 10.2 修改范围

```text
ui/widgets/settings_dialog.py
ui/widgets/batch_dialog.py
ui/main_window.py
相关 import
相关测试
相关文档
```

### 10.3 执行步骤

1. 确认新全局设置可用。
2. 确认新设备设置可用。
3. 确认新批量设置可用。
4. 旧 `settings_dialog.py` 改为兼容提示或删除。
5. 旧 `batch_dialog.py` 改为兼容提示或删除。
6. 清理无用 import。
7. 更新文档。
8. 更新测试。

### 10.4 验收标准

```text
- [ ] 点击全局设置只打开全局设置。
- [ ] 双击设备只打开设备设置。
- [ ] 批量设置按钮只打开批量设置。
- [ ] 旧混合弹窗不再作为主入口。
- [ ] 代码中没有重复入口导致用户混淆。
```

---

## 11. 测试计划

### 11.1 静态检查

```bash
python -m compileall -q .
pytest -q
```

### 11.2 UI 手工测试清单

```text
- 启动 PCControl。
- 登录或进入主窗口。
- 进入设备管理页。
- 检查底部工具栏。
- 检查右侧侧栏。
- 搜索设备。
- 筛选设备。
- 全选、反选、清空选择。
- 打开设备设置。
- 打开批量设置。
- 打开全局设置。
- 确认三个入口不同。
```

### 11.3 密码测试清单

```text
- 新增一条游戏账号。
- 输入真实密码。
- 确认默认显示 ******。
- 右键显示密码。
- 右键隐藏密码。
- 右键复制密码。
- 编辑密码。
- 编辑完成自动隐藏。
- 全部显示。
- 全部隐藏。
- 脱敏导出。
- 明文导出二次确认。
- 检查日志无真实密码。
```

### 11.4 边界测试清单

```text
- 全局设置中不能看到游戏账号表。
- 全局设置中不能看到游戏任务路线配置。
- 全局设置中不能看到物品/交易/制造/铸币页。
- 设备设置中可以看到游戏账号设置。
- 批量设置中可以看到批量账号设置。
- 所有账号来源只有手动输入和客户外部账号数据库。
- UI 中不能出现远控/投屏入口。
```

---

## 12. 回退总方案

### 12.1 每阶段必须可独立回退

每个阶段必须单独提交，确保可以：

```text
git revert <commit_sha>
```

### 12.2 高风险阶段回退策略

#### P1 设备页重构失败

```text
临时恢复 main_window.py 旧设备页。
保留新 DevicePage 文件但不启用。
```

#### P2 全局设置失败

```text
主入口切回旧 settings_dialog.py。
新 GlobalSettingsDialog 保留但不启用。
```

#### P3 设备设置失败

```text
设备双击入口回退为旧行为或禁用。
DeviceSettingsDialog 保留但不启用。
```

#### P4 密码行为失败

```text
回退密码 delegate。
临时改为只隐藏显示，不允许明文导出。
保留数据模型不变。
```

#### P5 批量设置失败

```text
批量设置按钮切回旧 batch_dialog.py。
新 BatchSettingsDialog 保留但不启用。
```

---

## 13. 交付物清单

### 13.1 文档交付

```text
HiveGreatSage-Knowledge/02-PC中控框架/UI边界重构方案.md
HiveGreatSage-Knowledge/02-PC中控框架/UI边界重构执行方案.md
HiveGreatSage-PCControl/docs/pc_control/UI_BOUNDARY.md
```

### 13.2 代码交付

```text
ui/pages/device_page.py
ui/widgets/device_bottom_toolbar.py
ui/widgets/device_side_panel.py
ui/dialogs/global_settings_dialog.py
ui/dialogs/device_settings_dialog.py
ui/dialogs/batch_settings_dialog.py
core/settings/*.py
core/external_account_db/*.py
core/device_group/*.py
```

### 13.3 测试交付

```text
tests/ui/test_device_page_boundary.py
tests/ui/test_global_settings_boundary.py
tests/ui/test_account_password_behavior.py
tests/ui/test_batch_settings_boundary.py
```

测试文件名为建议命名，实际是否采用需根据当前测试结构确认。

---

## 14. 最终执行顺序摘要

```text
P0：冻结边界，写 UI_BOUNDARY.md，旧 settings_dialog.py 不再新增游戏配置。
P1：设备页拆分，底部工具栏，右侧侧栏。
P2：新全局设置，只保留平台附加能力和客户环境。
P3：新设备设置，承载单设备游戏配置。
P4：账号设置页，密码显示/隐藏/复制/编辑/导出规则。
P5：新批量设置，字段级勾选、差异预览、应用结果。
P6：客户外部账号数据库连接配置。
P7：清理旧混合设置入口。
P8：补测试、验收、回退文档。
```

---

## 15. 最终结论

本执行方案的核心是：

```text
先冻结边界，再拆 UI；
先拆入口，再迁移配置；
先做草稿和本地模型，再接后端接口；
先保证不失真，再追求完整功能；
每阶段都可回退，每阶段都有验收；
不引入远控/投屏；
不引入无证据账号来源；
不把游戏配置放进全局设置。
```
