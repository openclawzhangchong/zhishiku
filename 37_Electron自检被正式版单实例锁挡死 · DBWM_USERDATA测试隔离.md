# 37 · Electron 测试隔离：e2e 被正式版单实例锁挡死 → DBWM_USERDATA

> **摘要**：桌面应用的 e2e 自检脚本在「正式版正在托盘常驻」的真机上 4 组断言全部超时，
> 看着像捕获链路坏了，实际是开发进程被 `requestSingleInstanceLock` 挡掉秒退——
> 开发实例和正式版共用同一个 userData（Electron 按 productName 推导）。
> 反直觉点：**单实例锁不是防用户的，它同样防着你的自动化测试**；
> 而锁的粒度是 userData 路径，所以解法是给测试进程一份独立 userData，而不是杀正式版。

**关键词**：Electron、requestSingleInstanceLock、单实例锁、userData、e2e 自检、托盘常驻、app.setPath、秒退、测试隔离、DBWM_USERDATA

---

## 事故 1：e2e 全超时，「日志节选」为空

| 项 | 内容 |
|---|---|
| **症状** | `npm run e2e` 4 组断言（CDP 响应体 / 页面钩子 / 图片入库 / 视频候选）全部超时；「日志节选」整段为空——应用的运行日志一行都没有 |
| **错误归因** | 先怀疑自己改坏了捕获链路 → `git stash` 回干净 HEAD 复测，**基线同样失败**，排除代码问题；又怀疑 e2e 准备环境时 `unlinkSync captures.json` 的 ENOENT（日志里确有一行），但那行本来就有 try/catch 不是致命点 |
| **证据链** | ① 日志文件完全没被创建 → 应用根本没跑到 logger 初始化 ② 手动 `electron .` 启动，控制台报 `Unable to move the cache: 拒绝访问(0x5)` 后进程消失 ③ `tasklist` 查到**正式版 `豆包无水印下载器.exe` 正在托盘常驻** ④ Electron 的 userData 由 productName 推导，开发实例与正式版同为 `Roaming\豆包无水印下载器` ⑤ `requestSingleInstanceLock()` 拿不到锁 → `app.quit()` 秒退，无任何输出 |
| **根因** | 开发/自检进程与托盘常驻的正式版**共用 userData**，单实例锁把测试进程挡掉秒退；e2e 的 4 组超时全是这条根因的下游症状 |
| **为什么没早发现** | 写 e2e 的机器上正式版恰好没在跑；「自检依赖真机、真机天天挂着正式版」这个组合只在日常使用的机器上出现 |
| **修复** | main.js 模块顶部：`if (process.env.DBWM_USERDATA) app.setPath('userData', process.env.DBWM_USERDATA)`（放在 `requestSingleInstanceLock` **之前**，锁按 userData 推导）；e2e 脚本把 settings/captures 全部写进 `fs.mkdtempSync` 临时目录并以 `DBWM_USERDATA` 注入，结束时 `rmSync` 清理。正常用户不设这个变量，行为零变化 |
| **可复用教训** | Electron 应用的 e2e/自动化**必须走独立 userData**（在 `requestSingleInstanceLock` 之前 `app.setPath('userData', ...)`），否则任何一台「正式版托盘常驻」的真机上，测试进程都会被单实例锁静默秒退，表现成所有断言超时 |

**验证状态**：真机实测通过（正式版托盘常驻的情况下 e2e 4 组断言全绿）

> 关联：`30_Windows打包`（同项目）、`01_AI协作`。

## 顺带的第二个坑：Git Bash 会吃掉 NSIS 的 `/S` 参数

| 项 | 内容 |
|---|---|
| **症状** | Git Bash 里 `setup.exe /S` 静默安装不生效，弹出交互式安装窗口 |
| **根因** | MSYS 对以 `/` 开头的参数做**路径转换**（`/S` → `S:\`），NSIS 收不到静默开关 |
| **修复** | 用 PowerShell `Start-Process -FilePath ... -ArgumentList '/S'`，或 Git Bash 里写 `//S` |
| **可复用教训** | 在 Git Bash 里调 Windows 原生安装器/工具，凡单斜杠开头的开关都要防 MSYS 路径转换 |

**验证状态**：真机实测通过（PowerShell `/S` 静默升级 Program Files 旧版成功）
