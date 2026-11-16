# 41 · 单文件安装包打包：bat 编码裂行 / `\f` 转义吃字符 / NSIS 树杀 / IExpress /Q

> **摘要**：给 PyInstaller 后端 + frpc 做一个"装完即用"的 NSIS 单文件安装包，四类坑全部踩中且都有解：① cmd 批处理 LF/UTF-8 裂行；② python 生成脚本时 `\f` 转义静默吃字符；③ IExpress /Q 不执行安装命令、NSIS 安装器退出带走子进程树；④ PyInstaller 冻结态路径必须环境变量显式指路。

**关键词**：NSIS, makensis, IExpress, /Q, QuietInstCmd, bat CRLF, bat UTF-8 裂行, chcp 65001, GBK 编码, 语句未结束, python 转义, \f 换页符, frprpc, Form Feed, start /min, 进程树, Job Object, cmd /c start 分离, PyInstaller onefile, 冻结态, ROOT_DIR, _MEI, ZCODE_DATA_DIR, Expand-Archive, 7z a -tzip, 长路径

---

## 事故/主题 1：cmd 批处理 LF + UTF-8 中文 → 括号块解析裂行

| 项 | 内容 |
|---|---|
| **症状** | IExpress 静默安装静默失败；手动跑 bat 报 `'安装目录:' 不是内部或外部命令`、`'chive' 不是...`（Expand-Archive 被腰斩） |
| **证据链** | ① bat 由工具生成 = LF 换行 + UTF-8；② cmd 对 LF-only bat 的括号块解析错位，中文行被拆成命令执行；③ 转 CRLF 后解析正常 |
| **根因** | cmd 批处理要求 CRLF；LF + 括号块/中文在解析层裂行 |
| **修复** | 所有 bat 一律 CRLF；含中文的 bat 存 **GBK（ANSI）且去掉 chcp**（中文系统原生解析最稳），纯 ASCII 的才敢用 UTF-8 |
| **可复用教训** | 工具生成的 bat 交付前必转 CRLF；含中文的 bat 用 GBK 编码 + 不写 chcp，纯 ASCII 内容才考虑 UTF-8 |

**验证状态**：真机实测通过（转换后同脚本完整跑通安装流程）

---

## 事故/主题 2：python 生成 bat，`\f` 转义把 `frp\frpc` 吃成换页符

| 项 | 内容 |
|---|---|
| **症状** | bat 里 `start "%~dp0frprpc.exe"` —— 文件名损坏，frpc 永远启动不了；cat/查看时完全隐形 |
| **证据链** | ① 逐字节断言 `chr(12) not in t` + `'frprpc' not in t` 才暴露；② python 字符串 `"\f"` = Form Feed（合法转义），写入文件后 cat/编辑器里不可见 |
| **根因** | 用 python 字符串拼接生成代码文件时，`\f \b \v` 等合法转义被解释 |
| **修复** | 生成代码文件的 python 脚本：内容先按行列表写、或统一用原始字符串；交付前加断言（无控制字符 + 关键路径逐字节比对） |
| **可复用教训** | 凡"用程序生成脚本/配置"的场景，生成后必须做 `chr(12)` 等控制字符断言，隐形损坏肉眼查不出 |

**验证状态**：真机实测通过（断言后重建，frpc 正常启动）

---

## 事故/主题 3：IExpress /Q 与 NSIS 安装器退出带走子进程树

| 项 | 内容 |
|---|---|
| **症状** | ① IExpress SED 配了 AppLaunched 但 /Q 静默跑完什么都没装；② NSIS /S 装完进程起来了，安装器一退进程全没 |
| **证据链** | ① IExpress：/Q 只解压，执行安装命令须 SED 配 `UserQuietInstCmd=install.bat`；② NSIS：`Exec 'wscript autostart.vbs'` 派生的 backend/frpc 在安装器退出时整树消失；③ 改 `Exec '"$SYSDIR\cmd.exe" /c start "" /min ...'` 分离启动后进程存活 |
| **根因** | IExpress /Q 依赖 QuietInstCmd 配置；NSIS 退出时回收其直接派生的进程树（cmd start 可脱离） |
| **修复** | IExpress 要静默须配 QuietInstCmd 且被调脚本不能有 `pause`（改 `ping -n 4 127.0.0.1 >nul`）；NSIS 启动常驻服务用 `cmd /c start "" /min` 分离 |
| **可复用教训** | NSIS 里启动常驻服务必须经 `cmd /c start` 分离进程树；IExpress 静默安装必须配 QuietInstCmd 且脚本无交互暂停 |

**验证状态**：真机实测通过（分离启动后 backend/frpc 跨安装器退出存活）

---

## 事故/主题 4：PyInstaller 冻结态路径全变——三环境变量指路

| 项 | 内容 |
|---|---|
| **症状** | 后端 exe 换目录启动即崩（`app\statics does not exist`）；或进程活着但永远不监听端口 |
| **证据链** | ① `ROOT_DIR = Path(__file__).parents[1]` 冻结态 = `_MEI` 临时目录；② data/frontend/captcha_node 默认全落在临时目录；③ 用 `ZCODE_DATA_DIR / ZCODE_FRONTEND_DIR / ZCODE_CAPTCHA_SOLVER_DIR` 三环境变量显式指路后一切正常 |
| **根因** | PyInstaller onefile 每次解包到新临时目录，相对路径全失效 |
| **修复** | 启动器（bat/vbs）显式设置三个环境变量；或 `ZCODE_DATA_DIR` 指向固定 AppData 目录（换目录重装账号数据不丢） |
| **可复用教训** | PyInstaller 冻结应用的可变数据/资源一律用环境变量显式指路，装到固定目录（如 LOCALAPPDATA）+ 数据放 AppData |

**验证状态**：真机实测通过

---

## 可复用教训速查

- bat = CRLF 必须；含中文 → GBK 编码 + 不写 chcp
- 程序生成脚本 → 生成后断言无控制字符（`\f` 隐形损坏）
- NSIS 启动常驻服务 → `cmd /c start "" /min` 分离进程树
- IExpress 静默 → QuietInstCmd + 无 pause
- PyInstaller 打包的后端 → 三环境变量指路（data/frontend/captcha），数据放 AppData
- 本机大文件 zip 用 7z（`7z a -tzip`），长路径稳；单文件安装 exe 用 makensis（Tauri 自带 NSIS 可复用）

> 关联：`40_frp复用上线公网隧道`、`29_GitHub访问`（API 直传 push）。


---

## 补充实测（2026-11-16 打包当日追加，同一交付中发现）

### 补1：tasklist //FI 经 Git Bash 间歇失效 → 误判「进程死了」

| 项 | 内容 |
|---|---|
| **症状** | tasklist //FI "IMAGENAME eq xxx.exe" 时而正常时而返回「没有运行的任务」，服务明明活着（公网 200 实证） |
| **根因** | MSYS 对 //FI 参数的转换/传递间歇异常 |
| **修复** | 进程存在性检查改用 powershell Get-Process 或 netstat -ano 交叉验证 |
| **可复用教训** | Git Bash 下 tasklist //FI 的否定结论不可信，必须用 PowerShell/netstat 复核后再下判断 |

### 补2：PyInstaller --noconsole exe 在隐藏窗口启动会卡死（不监听端口）

| 项 | 内容 |
|---|---|
| **症状** | --noconsole 打包的后端 exe：前台/CREATE_NO_WINDOW+DEVNULL 启动 2 秒即监听；但 vbs 隐藏启动（无重定向）进程活着却永远不 bind 端口 |
| **根因** | --noconsole 应用 stdout 无效；uvicorn 日志写 stdout 在某些句柄形态下卡死启动流程 |
| **修复** | 隐藏启动必须重定向 stdout：cmd /c "exe" >> "log" 2>&1（文件句柄有效）或 python Popen DEVNULL |
| **可复用教训** | --noconsole 后端做无窗口自启时，stdout 必须显式重定向到文件，不能裸启动 |

### 补3：Git Bash 里 >nul 会创建名为 nul 的真实文件，卡死 git add

| 项 | 内容 |
|---|---|
| **症状** | git add -A 报 error: unable to index file 'nul'，提交失败 |
| **根因** | bash 下 >nul 不走 Windows NUL 设备，而是创建保留名文件 nul；git 无法索引 |
| **修复** | 删除用 rm "//?/<绝对路径>/nul" 或 PowerShell Remove-Item -LiteralPath "\\?\...nul"；bash 里重定向一律用 /dev/null |
| **可复用教训** | Windows 保留名文件（nul/con）用 \\?\ 前缀删；在 Git Bash 里永远写 /dev/null 不写 nul |

### 补4：build.bat 的 move /y 在 Git Bash 不存在 → 拷贝到安装包的是旧产物

| 项 | 内容 |
|---|---|
| **症状** | 重编后端后安装包里仍是旧版本（新 exe 尺寸 28296341 ≠ 安装的 28296028） |
| **根因** | build.bat 用 move /y（Windows 命令），Git Bash 无 move → 改名步骤静默失败，后续拷贝的是旧文件 |
| **修复** | bash 下用 mv；或整条构建链都从 cmd 跑；交付前核对产物尺寸/mtime |
| **可复用教训** | bat 里的 Windows 专属命令（move/del nul）不能在 Git Bash 直接复跑，逐条翻译成 bash 等价物 |
