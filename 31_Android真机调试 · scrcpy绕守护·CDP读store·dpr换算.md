# 31 · Android真机调试：scrcpy绕守护、CDP读store、dpr换算

> **摘要**：Capacitor/WebView 应用真机调试的完整方法论，全部经 realme（Android 16）实战验证。三个核心手段：**scrcpy 投屏**（绕过 ColorOS「安全守护」对 `input tap` 的拦截，电脑上直接操作手机）；**WebView CDP**（比截图精准一个量级——可读 pinia store、可取任意元素 getBoundingClientRect 精确坐标）；**uiautomator dump**（拿物理坐标，避免"截图尺寸 ≠ 物理分辨率"的换算错误）。另含 6 种失败的安装方式记录与可用的 2 种。

**关键词**：Android 真机、scrcpy、WebView CDP、realme 安全守护、adb、pinia、devicePixelRatio、uiautomator

---

## 事故 1：adb install 卡死无输出——「看着像 adb 坏了，其实是安全守护在等确认」

| 项 | 内容 |
|---|---|
| **症状** | `adb install -r xx.apk` 无输出超时；`pm install` 也无输出 |
| **错误归因** | "adb 驱动/线/端口问题" |
| **证据链** | ① `adb devices` = device（已授权）② 截屏（`adb shell screencap`）看屏幕：停在 ColorOS「安全守护 · 存在以下风险（未经安全审核）」页，等人工确认 ③ 该页含「已知悉该应用存在风险」复选框 + 「授权本次安装」按钮 |
| **根因** | realme/ColorOS 对**首次安装的签名**强制人工过风险页；`adb shell input tap` 在该页被系统拦截（注入无效），所以连 adb 点击自动化都不行 |
| **为什么没早发现** | 逐个试了 6 种方案（install -r / -r -d / push+pm install / settings put verifier / am start intent / 勾选+授权点击）全卡在同一步 |
| **修复** | **方法 A**：手机上手动点（勾选→授权）**方法 B（电脑可操作）**：scrcpy 投屏后用鼠标点 |
| **可复用教训** | 首次安装卡住先截屏（`adb shell screencap`）看是不是停在系统拦截页；**同签名装过一次后，`adb install -r` 直接成功**（风险确认是一次性的） |

---

## 方法论 1：scrcpy 投屏（绕过 tap 拦截的正规路径）

```bash
curl -L https://github.com/Genymobile/scrcpy/releases/download/v5.0.1/scrcpy-win64-v5.0.1.zip -o scrcpy.zip
python -c "import zipfile; zipfile.ZipFile('scrcpy.zip').extractall('scrcpy')"
cd scrcpy/scrcpy-win64-v5.0.1
./scrcpy.exe --window-title="手机控制" --stay-awake
```

- 窗口 = 手机实时屏幕，**鼠标点击/输入直接注入**（独立通道，实测在安全守护页可用）
- 5.0.1 参数写法：`--turn-screen-off=false` 这种带值布尔会报错，用 `--no-turn-screen-off` 或省略
- `--window-title` 可自定义，方便识别窗口

---

## 方法论 2：WebView CDP（比截图精准一个量级）

Capacitor 应用 = Chromium WebView，天然支持 Chrome DevTools Protocol：

```bash
PID=$(adb shell pidof com.zcmusic.app | tr -d '\r')
adb forward tcp:9333 localabstract:webview_devtools_remote_$PID
curl http://127.0.0.1:9333/json     # 拿 webSocketDebuggerUrl
```

node 端 WebSocket 模式（`Runtime.evaluate`）可做三件截图做不到的事：

```js
// ① 读 pinia store（播放状态/当前歌/任意 state）
const app = document.querySelector('#app').__vue_app__;
const ps = app.config.globalProperties.$pinia._s.get('player');
ps.isPlay; ps.playMusic.name;

// ② 精确元素坐标（供 adb input tap 用）
const r = document.querySelector('.effect-btn').getBoundingClientRect();
tapX = Math.round(r.x + r.width/2) * devicePixelRatio;   // CSS × dpr = 物理坐标
tapY = Math.round(r.y + r.height/2) * devicePixelRatio;

// ③ 列出同类元素全部坐标（避免点错相邻按钮）
[...document.querySelectorAll('.side-button')].map((b,i)=>{ ... });
```

**CDP 的两个实测限制**：
1. 需要用户手势的函数（如 `playMusicEvent()`）在 CDP 里调用无效 → 用 `adb shell input tap <物理坐标>` 真实点击
2. App 重启后 PID 变化 → 需重新 `adb forward`；`adb forward --remove-all` 先清理

---

## 事故 2：input tap 坐标全偏——「看着像坐标算错，其实是截图 ≠ 物理分辨率」

| 项 | 内容 |
|---|---|
| **症状** | 按截图目测的点 tap 多次无反应 |
| **证据链** | 截图 900×2000；`adb shell wm size` → **1080×2400**；uiautomator dump 出的 bounds 是物理坐标 |
| **根因** | 用截图尺寸算坐标（比例 2400/2000=1.2 还算对了一半，x 轴 900→1080 漏算） |
| **修复** | 一律用 `uiautomator dump` 拿 bounds 中心，不用截图目测 |
| **可复用教训** | **截图分辨率 ≠ 物理分辨率**。tap 坐标只认 uiautomator 的 bounds 或 `getBoundingClientRect × dpr` |

---

## 方法论 3：截图链路（Windows 侧中文路径坑）

```bash
# 手机截图（adb）
adb shell screencap -p /sdcard/sc.png && adb pull /sdcard/sc.png C:\Users\<user>\sc.png

# Windows 桌面截图（PowerShell System.Drawing）
Add-Type -AssemblyName System.Windows.Forms,System.Drawing
$b = New-Object System.Drawing.Bitmap([Windows.Forms.Screen]::PrimaryScreen.Bounds.Width, ...)
$g = [System.Drawing.Graphics]::FromImage($b)
$g.CopyFromScreen(0,0,0,0,$b.Size)
$b.Save("C:\Users\<user>\shot.png")
```

**坑**：保存路径含中文（`E:\github项目\`）时 PowerShell 内联命令转义极易出错（`\U` 被当 unicode 转义、引号嵌套炸掉）。**先写 .ps1 文件再 `-File` 执行**，输出路径用 `C:\Users\<user>\`。

---

## 方法论 4：版本/安装验证速查

```bash
adb shell dumpsys package com.zcmusic.app | grep -E "versionName|versionCode"
adb shell pidof com.zcmusic.app
adb shell am start -n com.zcmusic.app/.MainActivity
adb shell am force-stop com.zcmusic.app
adb shell pm list packages | grep zcmusic
aapt2 dump badging app-release.apk | grep -E "versionCode|application-label"
```

---

## 一页速查

| 场景 | 用什么 |
|---|---|
| 装不上卡在风险页 | 手机手点 / scrcpy 投屏点（同签名装过一次后 adb install -r 直接过） |
| 想知道真实 UI 状态 | `adb shell screencap` + `adb pull` |
| 想读运行时状态（store/变量） | WebView CDP `Runtime.evaluate` |
| 想点某个元素 | CDP 取 `getBoundingClientRect × dpr` → `adb shell input tap`（非拦截页） |
| 想拿准确按钮坐标 | `adb shell uiautomator dump`（bounds = 物理坐标） |
| 桌面操作手机 | scrcpy |

> 关联：`30_Windows打包 · build.files白名单`（打包出问题 App 本体崩）、`19_桌面自动化 · 无控件树`（另一套桌面自动化经验）、`29_GitHub访问`。
