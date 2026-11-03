---
id: 30
title: Windows打包
stack: ["electron", "electron-builder", "nsis"]
keywords: ["build.files", "asar", "EBUSY", "WinError 5", "UAC.dll_unloaded", "无法关闭", "Failed to uninstall"]
symptoms: ["装完双击没反应", "覆盖安装弹无法关闭", "安装包体积虚高", "asar缺入口"]
whenToRead: 打包前/覆盖安装前必读
oneLiner: build.files 必须白名单（.gitignore 会吃掉 out/）；事件日志的故障模块名会骗人；装到用户可写目录免提权
---

# 30 · Windows打包：build.files白名单、事件日志会骗人、安装器钩子

> **摘要**：Electron 项目把 `build.files` 配成 `["**/*"]` 通配符，而 `.gitignore` 里恰好有 `out`——electron-builder **遵循 .gitignore**，编译产物 `out/main/index.js` 被整体排除在 asar 之外，装完双击无反应。Windows 事件日志报 `UAC.dll_unloaded` + `0xc000041d`，把排查引向 UAC/沙盒/ACL 三个方向全错，耗时半天；真正定位只花 10 分钟：`npx asar list ... | grep -c "out\\"` = **0**。本篇同时沉淀覆盖安装「无法关闭」「Failed to uninstall old application files: 2」两个弹窗的真实机制与排查工具箱。

**关键词**：electron-builder、build.files 白名单、.gitignore 影响 asar、事件日志误导、NSIS 覆盖安装、安装目录智能推荐、低权/提权对照实验、发版 checklist

---

## 事故 1：装完双击没反应——「看着像 UAC，其实是 asar 缺入口」

| 项 | 内容 |
|---|---|
| **症状** | 安装成功但双击图标没反应/闪退；`--no-sandbox`、`--disable-gpu` 全试无效；事件日志故障模块 = `UAC.dll_unloaded` |
| **错误归因** | 按事件日志走：UAC 提权问题 → 沙盒组件拦截 → EDR —— 全错，耗时半天 |
| **证据链** | ① `npm run dev` 一切正常 → 代码没问题 ② `npx asar list dist/win-unpacked/resources/app.asar \| grep -c "out\\"` → **0**（入口文件根本没进包）③ asar 顶层全是 node_modules / android Gradle 缓存 / 日志文件 ④ `.gitignore` 第 23 行 = `out` ⑤ `build.files = ["**/*", ...]` |
| **根因** | electron-builder 打包**遵循 .gitignore**。`out` 被忽略 → 编译产物全排除；`**/*` 同时把 android/ 的 Gradle 缓存、node_modules 全量打进 asar（722MB） |
| **为什么没早发现** | 事件日志给了假线索；更早一版「恰好能跑」是当次构建时序差异，掩盖了配置错误 |
| **修复** | `build.files` 改白名单：`out/**/*` + `package.json`，显式排除 android/dist/docs/src/build 等；asar 722MB→116MB（-84%），安装包 513MB→156MB（-70%） |
| **可复用教训** | ① electron-builder **永远用白名单**列构建产物，不要 `**/*` ② `.gitignore` 会影响打包内容（想打包的产物目录绝不能进 .gitignore）③ **崩溃事件日志的"故障模块"可能是误导**，先验证产物完整性再追日志线索 |

**发版必检**：
```bash
npx asar list dist/win-unpacked/resources/app.asar | grep -c "out\\"   # >0
dist/win-unpacked/ZCMusic.exe   # 直接启动冒烟
node --check out/main/index.js
```

---

## 事故 2：覆盖安装弹「ZCMusic 无法关闭」——看着像进程占用，其实是删除被拦

| 项 | 内容 |
|---|---|
| **症状** | 覆盖安装进度 ~55% 弹「ZCMusic 无法关闭。请手动关闭它」（重试/取消）；任务管理器里**没有** ZCMusic.exe；点「重试」依旧失败；再点出现第二弹窗：**「Failed to uninstall old application files. Please try running the installer again. : 2」** |
| **错误归因** | "有进程占用" → 进程扫描 = 0；"文件被锁" → Sysinternals Handle 全盘扫 = 无句柄 |
| **证据链** | ① **对照实验**：同一批文件，低权 shell 删除 → WinError 5；UAC 提权 PowerShell 删除 → **成功**（19 个文件全清）② 提权运行旧卸载器 `Uninstall ZCMusic.exe /S /allusers _?=dir` → **exit 0**，卸载器本身正常 ③ 真正干扰源：第三方沙盒组件（sandbox-center.exe）监控 Program Files 下的应用文件 |
| **根因** | **安装器以低权令牌调用旧卸载器**（或提权链路被第三方组件干扰），删除 `Program Files` 下文件被拒 → 卸载器非 0 退出。另外「无法关闭」这个文案被 NSIS 模板**复用为删除失败提示**，与"进程占用"无关，极误导 |
| **为什么没早发现** | 两个弹窗文案都指向"进程占用"方向；直到做了低权/提权对照实验才转向 |
| **修复** | ① `customInit` 钩子里 `taskkill /F /IM "ZCMusic.exe" /T` 自动结束实例 ② `customUnInstallCheck` 钩子里 taskkill 干扰源（沙盒组件）后重试 ③ **安装目录智能推荐**：默认路径含 Program Files 时自动探测非系统盘（空闲>2GB）改为 `<盘>:\ZCMusic`，或退回 `%LOCALAPPDATA%\Programs\App` —— 装到用户可写目录后覆盖升级**免提权**，从根上免疫此类故障 |
| **可复用教训** | ① 「无法关闭」弹窗**先做低权/提权对照实验**，别急着杀进程 ② 桌面软件安装目录选用户可写位置，比依赖提权可靠 |

### 排查工具箱（按症状选用）

| 症状 | 方法 |
|---|---|
| 文件删不掉：区分"有锁"还是"权限" | `CreateFileW(path, GENERIC_READ, 0, None, OPEN_EXISTING)`：err=32 共享冲突（有锁）/ err=5 权限（UAC 提权或所有权问题） |
| 顽固进程杀不掉 | WMI：`Get-CimInstance Win32_Process -Filter "ProcessId=$pid" \| Invoke-CimMethod -MethodName Terminate`（taskkill/Stop-Process 都拒绝时） |
| 谁占着文件 | Sysinternals `Handle.exe -accepteula <关键字>`（可下载单个 exe） |
| 删除整个目录定位具体文件 | python `os.walk(topdown=False)` 逐个删除并统计失败清单 |
| 崩溃原因 | 事件查看器 / `C:\ProgramData\Microsoft\Windows\WER\ReportArchive\` 的 Report.wer——**故障模块名可能是误导**（如 UAC.dll_unloaded 实为 asar 缺入口） |
| ACL 检查 | `icacls <file>` / `Get-Acl`；注意继承标记 (I) 与 TrustedInstaller 拥有的目录 |
| 提权执行 | `Start-Process powershell -Verb RunAs -Wait -ArgumentList '-File xxx.ps1'`（输出写文件带回） |

### 关键区分实验（2026-11-02 实战）

同一批文件：低权 shell 删除 → WinError 5；**UAC 提权 PowerShell 删除 → 成功**。
→ 证明文件本身正常，是调用方令牌问题。**先做这个对照实验，再怀疑沙盒/杀软。**

### NSIS 自定义钩子速查（electron-builder）

| 钩子 | 时机 | 用法示例 |
|---|---|---|
| `customInit` | 安装器初始化（向导前，$INSTDIR 可改写作为默认值） | 版本检查、taskkill 旧实例、安装目录推荐 |
| `customUnInstallCheck` | 旧卸载器返回后（handleUninstallResult SHELL_CONTEXT 分支） | 清理干扰源后重试 |
| `customInstall` / `customPageAfterChangeDir` / `customRemoveFiles` | 见 templates/nsis/*.nsh | — |

---

## 事故 3：electron-updater 遗留隐患（未爆雷，先记账）

| 项 | 内容 |
|---|---|
| **症状** | App 启动即崩 + `%LOCALAPPDATA%\<app>-updater\installer.exe` 残留 513MB |
| **证据链** | ① `autoInstallOnAppQuit = true` 时退出会自动运行 pending 安装包 ② pending 包来自私仓 GitHub（匿名 404）或旧版本，版本错位时装旧版 |
| **根因** | 下载失败/中断的 pending 安装器不清理；版本比对缺失 |
| **修复** | 手动移除 pending installer；建议 `autoInstallOnAppQuit = false` 或加版本比对 |
| **可复用教训** | 启动即崩先查 `%LOCALAPPDATA%\<app>-updater\` 残留；自动更新 + 无证书 + 私仓 = 高危组合 |

---

## 附：Windows 打包发版 checklist

```
□ 版本三处同步（package.json / build.gradle versionCode+versionName）
□ npm run build（BUILD_TARGET 区分 android/desktop）
□ npx asar list 确认 out/main/index.js 在包里（>0）
□ win-unpacked/ZCMusic.exe 启动冒烟
□ aapt2 dump badging（APK：versionCode/versionName/label）
□ APK 真机 adb install -r 冒烟
□ CHANGELOG 条目 / README 版本表 / 下载页 LABELS 字典（如新增版本文件名需补充）
□ git push（被墙走隧道，见 29_GitHub访问）
```

> 关联：`29_GitHub访问 · ECS隧道三模式`（推送被墙）、`25_Windows静默后台任务`（后台任务策略）。
