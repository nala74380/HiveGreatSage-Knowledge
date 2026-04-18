---
状态: 定稿 v1
最后更新: 2026-04-18
关联对话: 03-安卓脚本框架-推演-0418
前置文档: "[[03-安卓脚本框架/架构设计]] v3"
tags:
  - 安卓脚本
  - UI设计
  - H5
  - 动态UI
  - 案例学习
---

# UI 技术方案 — 案例学习与框架应用

> **文档说明：** 本文档对两个案例源码进行深度分析，提炼核心机制。
> 不照搬原始代码（原案例依赖第三方 DDMControl 云控插件），
> 而是用懒人精灵原生 API **自行实现**等价功能，结合蜂巢·大圣框架需求落地。

---

## 一、两种 UI 方案总览

| 对比项 | 动态 UI（`ui.*` API） | H5 WebView |
|---|---|---|
| 实现方式 | 纯 Lua 代码动态创建控件 | HTML + Vue2 + Vant2，离线加载 |
| 文件依赖 | 无额外文件 | .html + JS库（放入 .rc） |
| `showUI` 行为 | **非阻塞**（`ui.show` 后立即继续执行） | **阻塞**（等 `bridge.confirm()` 才返回） |
| 倒计时方案 | `for循环 + sleep(1000)` 直接驱动 | `setTimer` 回调驱动（showUI阻塞期间） |
| 配置持久化 | `ui.saveProfile` / `ui.loadProfile` | `localStorage` + `window.bridge.writeStringToFile` |
| 适用场景 | 快速参数输入、启动倒计时、简单配置 | 复杂多参数、分组分页、需要美观布局 |
| 控件丰富度 | 中等（文本/输入框/按钮/下拉/单选/多选/Tab） | 丰富（Vant2 全控件库） |

---

## 二、动态 UI 方案

### 2.1 核心机制理解

**`ui.show()` 是非阻塞调用** — 这是理解动态 UI 的关键。
调用 `ui.show("名称")` 后，脚本立即继续往下执行，不会等待界面关闭。
这使得"倒计时 + 界面"可以在同一线程顺序写出来。

```
ui.show("主界面")          ← 立即返回，界面在后台显示
↓
for 倒计时循环              ← 主线程继续跑，每秒更新按钮文字
↓
while 关闭UI==false do     ← 阻塞等待用户点击关闭或倒计时归零
  sleep(100)
end
↓
继续执行主程序逻辑
```

**关闭检测用全局标志变量：**

```lua
关闭UI = false

function onClose()
    ui.saveProfile("/sdcard/config.json")  -- 保存配置
    关闭UI = true
    ui.dismiss("主界面")
end
ui.setOnClose("主界面", "onClose()")

-- 等待关闭
while 关闭UI == false do sleep(100) end
-- 执行到这里说明界面已关闭
```

### 2.2 倒计时实现（原案例核心逻辑提炼）

原案例实现方式：

```lua
自动运行 = true

function 暂停倒计时()
    ui.setButton("按钮_倒计时", "已暂停自动运行")
    自动运行 = false
end
ui.setOnClick("按钮_倒计时", "暂停倒计时()")

-- 先 show，再跑倒计时循环 — 非阻塞的优势
ui.show("主界面")

local 等待秒数 = 60
for i = 1, 等待秒数 do
    if 自动运行 == false then break end
    if i == 等待秒数 then onClose() end  -- 归零自动关闭
    local 文字 = string.format("%d秒后自动运行(点击暂停)", 等待秒数 - i)
    ui.setButton("按钮_倒计时", 文字)
    sleep(1000)
end

while 关闭UI == false do sleep(100) end
```

**⚠️ 关键理解：** 倒计时必须用 `for循环 + sleep(1000)`，不能用 `setTimer`。
因为 `ui.show()` 不阻塞，主线程可以直接跑循环。
`setTimer` 只能用于 H5 的 `showUI`（阻塞调用）期间。

### 2.3 配置持久化（不依赖第三方插件）

原案例用 DDMControl 云配置，我们改用本地文件：

```lua
-- 保存配置（界面关闭时）
function on_close()
    ui.saveProfile("/sdcard/hive_config.json")
    ui.dismiss("主界面")
end

-- 恢复配置（界面显示前）
pcall(function()
    ui.loadProfile("/sdcard/hive_config.json")
end)
-- 用 pcall 包裹防止文件不存在时报错

ui.show("主界面")
```

**多配置方案（本地实现，替代原案例的云配置）：**

```lua
-- 配置存储路径格式：/sdcard/hive_config_方案名.json
local CONFIG_DIR = getSdPath() .. "/hive_configs/"

-- 获取所有方案列表
function get_config_list()
    local list = {}
    for f in lfs.dir(CONFIG_DIR) do
        if string.sub(f, -5) == ".json" then
            local name = string.sub(f, 1, -6)  -- 去掉 .json
            table.insert(list, name)
        end
    end
    return list
end

-- 保存当前方案
function save_config(name)
    local data = ui.getData()
    data["_meta_scheme_name"] = nil  -- 不保存下拉框本身
    lfs.mkdir(CONFIG_DIR)
    writeFile(CONFIG_DIR .. name .. ".json", jsonLib.encode(data))
    toast("方案已保存：" .. name, 0, 0, 12)
end

-- 加载方案
function load_config(name)
    local path = CONFIG_DIR .. name .. ".json"
    if not fileExist(path) then return end
    pcall(function()
        ui.loadProfile(path)
    end)
end
```

### 2.4 完整动态 UI 登录页模板（框架实现）

结合 [[03-安卓脚本框架/架构设计]] 的 `main.ui` 需求：

```lua
-- ============================================================
-- game/ui_helper.lua — 启动 UI（登录页 + 设置页）
-- 动态 UI 方案，不依赖第三方插件
-- ============================================================
local Config    = require("config")
local CloudSync = require("framework/cloud_sync")

local UIHelper = {}

local _close_flag   = false    -- 全局关闭标志
local _do_login     = false    -- 用户点了"登录"
local _countdown    = 0        -- 当前倒计时秒数
local _auto_run     = true     -- 是否允许自动倒计时

-- 倒计时秒数（初次/二次 由调用方决定）
function UIHelper.show_startup_ui(is_first_login)
    _countdown  = is_first_login
                  and Config.LOGIN_COUNTDOWN_INITIAL
                  or  Config.LOGIN_COUNTDOWN_RELOGIN
    _close_flag = false
    _do_login   = false
    _auto_run   = true

    local UI = "蜂巢启动界面"
    _build_ui(UI)
    _restore_config()     -- 恢复上次填写的账号（不含密码）
    ui.show(UI, false)    -- false = 不显示底部确认/退出按钮

    -- 非阻塞后立即跑倒计时
    for i = 1, _countdown do
        if not _auto_run or _close_flag then break end
        if i == _countdown then
            -- 倒计时结束，触发自动登录
            _do_login = true
            _close_flag = true
            ui.dismiss(UI)
            break
        end
        local remain = _countdown - i
        ui.setButton("btn_countdown",
            string.format("%ds 后自动登录（点击暂停）", remain))
        sleep(1000)
    end

    -- 等待用户操作或倒计时归零
    while not _close_flag do sleep(100) end

    -- 返回结果
    return _do_login, _get_form_data()
end

-- 构建 UI 布局
function _build_ui(name)
    ui.newLayout(name, -1, -1)

    -- ===== 顶部 Logo 区 =====
    ui.addTextView(name, "tv_title", "🐝 蜂巢·大圣")
    ui.newRow(name, "row_sep0")

    -- ===== Tab 页：登录 / 设置 =====
    ui.addTabView(name, "main_tab")
    ui.addTab("main_tab", "tab_login",    "登录")
    ui.addTab("main_tab", "tab_settings", "设置")

    -- ── 登录 Tab ──
    ui.addTextView("tab_login", "lbl_account", "账号")
    ui.addEditText("tab_login", "edit_account", "", -1)
    ui.newRow("tab_login", "row_l1")
    ui.addTextView("tab_login", "lbl_password", "密码")
    ui.addEditText("tab_login", "edit_password", "", -1)
    ui.newRow("tab_login", "row_l2")
    ui.addTextView("tab_login", "lbl_device", "编号（选填）")
    ui.addEditText("tab_login", "edit_device", "", -1)
    ui.newRow("tab_login", "row_l3")

    -- 倒计时按钮（点击暂停）
    ui.addButton("tab_login", "btn_countdown", "初始化中...")
    ui.setOnClick("btn_countdown", "_on_pause_countdown()")
    ui.newRow("tab_login", "row_l4")

    -- 随机延时勾选
    local delay_enabled = Config.RANDOM_DELAY_ENABLED
    ui.addCheckBox("tab_login", "cb_random_delay",
        string.format("登录后随机延时 %d–%ds（多设备防同步）",
        Config.RANDOM_DELAY_MIN, Config.RANDOM_DELAY_MAX),
        delay_enabled)
    ui.newRow("tab_login", "row_l5")

    -- 取消 / 登录 按钮行
    ui.addButton("tab_login", "btn_cancel", "取消")
    ui.addButton("tab_login", "btn_login",  "登录")
    ui.setOnClick("btn_cancel", "_on_cancel()")
    ui.setOnClick("btn_login",  "_on_login()")

    -- ── 设置 Tab ──
    ui.addTextView("tab_settings", "lbl_ctrl_ip", "中控 IP:端口")
    ui.addEditText("tab_settings", "edit_ctrl_ip",
        readKeyVal("ctrl_ip") or "", -1)
    ui.addTextView("tab_settings", "lbl_ctrl_tip",
        "PC中控局域网IP，用于组队/任务调度指令")
    ui.newRow("tab_settings", "row_s1")

    -- 当前代理（只读，来自云端下发）
    local proxy_name = "（未配置代理）"
    local ok, Proxy = pcall(require, "framework/proxy")
    if ok then proxy_name = Proxy.current_name() end
    ui.addTextView("tab_settings", "tv_proxy",
        "当前代理：" .. proxy_name)
    ui.addTextView("tab_settings", "tv_proxy_tip",
        "代理配置由PC中控下发，设备端仅展示")
    ui.newRow("tab_settings", "row_s2")

    -- 危险区：全局初始化
    ui.addTextView("tab_settings", "tv_danger_title", "⚠ 全局初始化")
    ui.addTextView("tab_settings", "tv_danger_desc",
        "清除游戏历史数据，适用于封禁后换号重启。操作不可撤销！")
    ui.newRow("tab_settings", "row_s3")
    ui.addButton("tab_settings", "btn_global_init", "执行全局初始化")
    ui.setOnClick("btn_global_init", "_on_global_init()")
    ui.newRow("tab_settings", "row_s4")
    ui.addButton("tab_settings", "btn_save_settings", "保存设置")
    ui.setOnClick("btn_save_settings", "_on_save_settings()")

    -- 关闭回调
    ui.setOnClose(name, "_on_ui_close()")
end

-- 恢复配置（密码不保存）
function _restore_config()
    local saved_account = readKeyVal("last_account") or ""
    if saved_account ~= "" then
        -- ui.loadProfile 会恢复所有字段，但密码框需要特殊处理
        pcall(function()
            ui.loadProfile(getSdPath() .. "/hive_ui.json")
        end)
        -- 清空密码（安全起见，密码不持久化）
        ui.setText("edit_password", "")
    end
end

-- 获取表单数据
function _get_form_data()
    local ok, data = pcall(ui.getData)
    return ok and data or {}
end

-- ── 回调函数（全局，供 setOnClick 字符串调用）──

function _on_pause_countdown()
    _auto_run = false
    ui.setButton("btn_countdown", "⏸ 已暂停（将在登录/取消时继续）")
end

function _on_login()
    local data = _get_form_data()
    -- 保存账号（不保存密码）
    if data.edit_account and data.edit_account ~= "" then
        writeKeyVal("last_account", data.edit_account)
    end
    ui.saveProfile(getSdPath() .. "/hive_ui.json")
    _do_login   = true
    _close_flag = true
    ui.dismiss("蜂巢启动界面")
end

function _on_cancel()
    _do_login   = false
    _close_flag = true
    ui.dismiss("蜂巢启动界面")
end

function _on_save_settings()
    local data = _get_form_data()
    -- 中控 IP 本地保存（不经云端）
    local ctrl_ip = data.edit_ctrl_ip or ""
    writeKeyVal("ctrl_ip", ctrl_ip)
    toast("设置已保存", 0, 0, 12)
end

function _on_global_init()
    -- 二次确认（用新建小布局实现弹窗）
    local dialog = "确认对话框"
    ui.newLayout(dialog, -2, -2)
    ui.addTextView(dialog, "tv_confirm",
        "⚠ 此操作将清除游戏全部本地数据！\n确认执行全局初始化？")
    ui.newRow(dialog, "dialog_row")
    ui.addButton(dialog, "btn_confirm_yes", "确认执行")
    ui.addButton(dialog, "btn_confirm_no",  "取消")
    ui.setOnClick("btn_confirm_yes", "_do_global_init()")
    ui.setOnClick("btn_confirm_no",  function() ui.dismiss(dialog) end)
    ui.show(dialog, false)
end

function _do_global_init()
    ui.dismiss("确认对话框")
    toast("正在执行全局初始化...", 0, 0, 12)
    -- 清除游戏数据
    exec("pm clear " .. Config.GAME_PACKAGE, false)
    -- 清除脚本运行时记录
    writeKeyVal("last_login_time", "")
    writeKeyVal("last_account", "")
    -- 游戏定制层可以在 on_global_init_hook() 里追加清除逻辑
    pcall(function()
        local game_ui = require("game/ui_helper")
        if game_ui.on_global_init then game_ui.on_global_init() end
    end)
    toast("全局初始化完成，脚本将重启", 0, 0, 12)
    sleep(2000)
    restartScript()
end

function _on_ui_close()
    _close_flag = true
    ui.saveProfile(getSdPath() .. "/hive_ui.json")
end

return UIHelper
```

---

## 三、H5 WebView 方案

### 3.1 核心机制理解

H5 方案的 `showUI` 是**阻塞调用**，等价于把一个 WebView 全屏展示，
直到 JS 端调用 `window.bridge.confirm()` 才返回。

**双向通信机制（两条通道）：**

```
H5 → Lua（写文件 + confirm）：
  window.bridge.writeStringToFile("/sdcard/config.json", JSON.stringify(data))
  window.bridge.confirm()   ← 触发 Lua 的 showUI 返回

Lua → H5（无直接通道）：
  只能在 showUI 调用之前把数据写入文件
  H5 在 mounted() 生命周期里读取 localStorage 恢复历史数据
```

**关闭检测（"结束运行"按钮）：**

```javascript
// H5 里的"结束运行"按钮
closeForm() {
    // 写入关闭标志
    window.bridge.writeStringToFile(
        "/sdcard/ui_close_flag.txt",
        '{"isCloseUI":true}'
    )
    window.bridge.confirm()
}
```

```lua
-- Lua 里检测是否是"关闭"操作
local function is_close_ui()
    local s = readFile("/sdcard/ui_close_flag.txt")
    if not s then return false end
    local ok, t = pcall(jsonLib.decode, s)
    return ok and t and t.isCloseUI == true
end

showUI("main.ui", -1, -1)   -- 阻塞

if is_close_ui() then
    exitScript()
end
local config = jsonLib.decode(readFile("/sdcard/config.json"))
```

**localStorage 历史配置恢复（H5侧）：**

```javascript
new Vue({
    data: {
        // 有历史记录则恢复，否则用默认值
        formData: localStorage.getItem("ui_data")
                  ? JSON.parse(localStorage.getItem("ui_data"))
                  : {}
    },
    methods: {
        submitForm() {
            this.$refs.form.getFormData().then(data => {
                const s = JSON.stringify(data)
                // 1. 写文件（给Lua读）
                window.bridge.writeStringToFile("/sdcard/config.json", s)
                // 2. 存 localStorage（下次H5自己恢复）
                localStorage.setItem("ui_data", s)
                // 3. 清除关闭标志
                window.bridge.writeStringToFile(
                    "/sdcard/ui_close_flag.txt", '{"isCloseUI":false}')
                // 4. 触发 Lua 的 showUI 返回
                window.bridge.confirm()
            })
        }
    }
})
```

### 3.2 我们的 H5 实现（不用 VmFormRender）

原案例用了 VmFormRender（一个 JSON 驱动的表单渲染库），配置在 `ui.js` 里。
我们**不用 VmFormRender**，直接写原生 Vue2 + Vant2 组件，
代码更清晰，控件语义更明确：

```html
<!-- 界面/h5/index.html（放入 .rc，extractAssets 后 WebView 加载）-->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1,
        maximum-scale=1,minimum-scale=1,user-scalable=no">
  <link rel="stylesheet" href="./vant212.min.css">
</head>
<body>
<div id="app">
  <!-- 顶部导航 -->
  <van-nav-bar title="蜂巢·大圣 — 参数配置" />

  <!-- Tab 分页 -->
  <van-tabs v-model="activeTab">
    <van-tab title="任务参数">
      <van-cell-group inset label="运行配置">
        <van-field v-model="form.task_count"
          label="任务数量" type="number" placeholder="请输入"/>
        <van-field v-model="form.task_interval"
          label="间隔(秒)" type="number" placeholder="请输入"/>
        <van-cell title="是否启用功能A">
          <template #right-icon>
            <van-switch v-model="form.enable_feature_a" size="20"/>
          </template>
        </van-cell>
      </van-cell-group>
    </van-tab>

    <van-tab title="账号设置">
      <van-cell-group inset>
        <van-field v-model="form.target_region"
          label="目标区服" placeholder="请输入区服名"/>
      </van-cell-group>
    </van-tab>
  </van-tabs>

  <!-- 底部按钮 -->
  <div style="padding:16px;display:flex;gap:12px">
    <van-button block type="info"   @click="onStart">开始运行</van-button>
    <van-button block type="danger" @click="onClose">结束退出</van-button>
  </div>
</div>

<script src="./vue2.6.14.min.js"></script>
<script src="./vant2.12.min.js"></script>
<script>
const STORAGE_KEY = "hive_h5_config"

new Vue({
  el: "#app",
  data() {
    // 从 localStorage 恢复历史配置
    const saved = localStorage.getItem(STORAGE_KEY)
    const defaults = {
      task_count:       "100",
      task_interval:    "5",
      enable_feature_a: true,
      target_region:    ""
    }
    return {
      activeTab: 0,
      form: saved ? Object.assign({}, defaults, JSON.parse(saved)) : defaults
    }
  },
  methods: {
    onStart() {
      const s = JSON.stringify(this.form)
      localStorage.setItem(STORAGE_KEY, s)
      window.bridge.writeStringToFile("/sdcard/config.json",    s)
      window.bridge.writeStringToFile("/sdcard/ui_close_flag.txt",
                                       '{"isCloseUI":false}')
      window.bridge.confirm()
    },
    onClose() {
      window.bridge.writeStringToFile("/sdcard/ui_close_flag.txt",
                                       '{"isCloseUI":true}')
      window.bridge.confirm()
    }
  }
})
</script>
</body>
</html>
```

**Lua 侧对接（H5方案）：**

```lua
-- game/ui_helper.lua — H5 方案（适用于复杂多参数界面）

local UIHelper = {}

function UIHelper.show_h5_ui()
    -- 1. 将 H5 文件从 .rc 释放到 sdcard（仅首次或版本更新时需要）
    local h5_dir = getSdPath() .. "/hive_h5/"
    if not fileExist(h5_dir .. "index.html") then
        lfs.mkdir(h5_dir)
        extractAssets("GameProject.rc", h5_dir, "*.html")
        extractAssets("GameProject.rc", h5_dir, "*.js")
        extractAssets("GameProject.rc", h5_dir, "*.css")
    end

    -- 2. 清除上次关闭标志
    writeFile(getSdPath() .. "/ui_close_flag.txt", '{"isCloseUI":false}')

    -- 3. 显示（showUI 阻塞，等 bridge.confirm() 返回）
    showUI("main.ui", -1, -1)

    -- 4. 检测是"开始"还是"关闭"
    local flag_s = readFile(getSdPath() .. "/ui_close_flag.txt") or ""
    local ok, flag = pcall(jsonLib.decode, flag_s)
    if ok and flag and flag.isCloseUI then
        return false, nil    -- 用户选择退出
    end

    -- 5. 读取配置
    local cfg_s = readFile(getSdPath() .. "/config.json") or "{}"
    local cfg_ok, config = pcall(jsonLib.decode, cfg_s)
    return true, (cfg_ok and config or {})
end

return UIHelper
```

**`.ui` 文件（懒人精灵界面文件）只需一个 WebView 控件：**

```xml
<窗口 宽度="-1" 高度="-1" 显示确认按钮="false" 显示标题栏="false">
  <标签页 标题="">
    <网页视图 id="webview" 地址="" 宽度="-1" 高度="-1"/>
  </标签页>
</窗口>
```

然后在 Lua 里，`showUI` 加载这个 .ui 文件，
通过 `setUIWebViewUrl` 把 H5 路径传进去：

```lua
-- 也可以在 onload 回调里动态设置 WebView 地址
showUI("main.ui", -1, -1, function(handle, event, a1, a2, a3)
    if event == "onload" then
        local url = "file://" .. getSdPath() .. "/hive_h5/index.html"
        setUIWebViewUrl(handle, 0, "webview", url)
    end
end)
```

---

## 四、方案选型决策（蜂巢·大圣框架）

```
main.ui（启动界面）：
  用动态 UI 方案
  原因：
  · 启动界面控件少（账号/密码/设备号/倒计时/中控IP/全局初始化）
  · 不需要额外 HTML 文件，简化部署
  · 动态 UI 的倒计时用 for循环 方案最简洁
  · showUI 非阻塞，倒计时代码和 UI 代码在同一函数里顺序写

游戏参数配置界面（game/ui_helper.lua 实现）：
  视游戏复杂度决定：
  · 参数 < 10 个：动态 UI，在 main.ui 的 Tab 里追加
  · 参数较多/需要分组分页：H5 方案，vue2 + vant2 原生组件
  · 不用 VmFormRender（避免依赖和学习成本）
```

---

## 五、TomatoOCR_SF.apk 与 YOLO 集成

### 5.1 已确认信息

```
APK 文件名：TomatoOCR_SF.apk（不是 TomatoOCR.apk）
加载方式：LuaEngine.loadApk("TomatoOCR_SF.apk")
类名：    com.tomato.ocr.lr.OCRApi

YOLO 模型：
  · 不需要单独加载 TomatoYOLO.apk
  · 直接把 YOLO 模型文件导入 .rc 资源包
  · 通过 TomatoOCR_SF 调用 YOLO 推理
  · 模型格式：待查阅官方文档确认（ncnn / onnx）
```

### 5.2 更新后的 `ocr.lua` 初始化代码

```lua
-- framework/ocr.lua（v1.1，修正 APK 文件名）

local loader = LuaEngine.loadApk("TomatoOCR_SF.apk")  -- ← 注意文件名
local OCRApi  = loader.loadClass("com.tomato.ocr.lr.OCRApi")
local engine  = OCRApi.init(LuaEngine.getContext())
engine.setLicense(Config.OCR_LICENSE)
-- ... 其余配置不变
```

### 5.3 YOLO 通过 TomatoOCR 调用（待补全）

```lua
-- ⚠️ 以下为框架预留，具体 API 待官网文档确认后补充
-- 已知：模型文件放入 .rc，不需要第二个 APK

-- 推测调用方式（待验证）：
-- engine.initYolo(modelPath)
-- local results = engine.detectYolo(bmp, threshold)

-- 占位接口（yolo.lua）：
local YOLO = {}
function YOLO.init(model_name)
    -- TODO: 从 .rc 释放模型文件，通过 TomatoOCR engine 加载
    print("[YOLO] 待实现：model=" .. model_name)
end
function YOLO.detect(l, t, r, b, threshold)
    -- TODO: 返回格式 [{label, x, y, w, h, score}]
    print("[YOLO] 待实现")
    return {}
end
return YOLO
```

---

## 六、关键 API 速查

### 动态 UI 常用 API

```lua
-- 布局
ui.newLayout(名称, w, h)
ui.newRow(布局名, 行id, [w], [h], [gid])
ui.addTabView(布局名, 控件名)
ui.addTab(tabViewName, 子Tab名, 标题)

-- 控件创建
ui.addTextView(父, id, 文字)
ui.addEditText(父, id, 默认值, [w], [h])
ui.addButton(父, id, 文字, [w], [h])
ui.addCheckBox(父, id, 文字, 是否勾选)
ui.addRadioGroup(父, id, {选项}, 默认index, [w], [h], 是否横向)
ui.addSpinner(父, id, {选项}, 默认index)

-- 显示/隐藏
ui.show(名称, [显示底部按钮], [x], [y], [显示标题])
ui.dismiss(名称)

-- 数据读写
ui.getData()                         → table（所有控件值）
ui.getText(id)                       → string
ui.setValue(id, val)
ui.setButton(id, 新文字)
ui.setSpinner(id, {新选项}, index)
ui.loadProfile(path)                 → 从JSON文件恢复控件值
ui.saveProfile(path)                 → 保存控件值到JSON文件

-- 事件绑定
ui.setOnClick(id, "函数名()")
ui.setOnClose(布局名, "函数名()")
ui.setOnChange(id, "函数名()")
```

### H5 通信 API

```javascript
// JS → Lua
window.bridge.writeStringToFile(path, content)  // 写文件
window.bridge.confirm()                          // 触发 showUI 返回
window.bridge.callLua("函数名(参数)")            // 直接调 Lua 函数
```

```lua
-- Lua 端
showUI("xxx.ui", w, h, callback)    -- 阻塞，等 bridge.confirm() 返回
-- 在 callback 的 onload 事件里：
setUIWebViewUrl(handle, pageIdx, idname, url)   -- 设置 WebView 地址
```

---

## 七、⚠️ 踩坑记录

```
动态 UI：
  · ui.show 非阻塞，倒计时必须用 for+sleep，不能用 setTimer
  · ui.loadProfile 如果文件不存在会报错，必须用 pcall 包裹
  · 多个 ui.setOnClick 的字符串参数是全局函数名，函数必须在全局作用域定义
  · ui.dismiss 之后 getData 会返回 nil，需要在 dismiss 前先取数据
  · Tab 里的控件事件直接写子Tab名作为父容器名

H5 方案：
  · showUI 阻塞期间，Lua 不能再调用 setTimer（回调在主线程，showUI 占着）
  · bridge.confirm() 只能调一次，调完 showUI 就返回了
  · localStorage 是 WebView 内部存储，重装 APK 后会清除
  · H5 文件要先 extractAssets 释放，不能直接读 .rc 内部路径

TomatoOCR：
  · APK 文件名是 TomatoOCR_SF.apk，不是 TomatoOCR.apk
  · YOLO 模型放 .rc，不需要第二个 APK
  · engine.setReturnType("json") 必须设置，否则返回格式不对
  · Bitmap 必须手动 recycle()，否则长时间运行必定内存溢出
```
