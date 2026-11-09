# 36 · 网络映射盘：Vite 解析失灵·本地 D1 fsync·cmd 中文注释炸解析

> **摘要**：项目放在 Windows 映射网络盘（`net use` 出来的 Z: = `\\服务器\共享\...`）上时，Vite 构建连 `vite` 自身都解析失败、wrangler 本地 D1 报 SQLite fsync IO 错、UTF-8 中文注释的 .cmd 批处理直接解析炸裂。三个坑根因不同（Vite realpath 盘符错位 / SQLite 要求本地 fsync / cmd 用 OEM 码页读脚本），但解法统一：**工作目录换成本地真路径（镜像同步），脚本纯 ASCII**。全部为 2026-11-08 个人工作台项目实测。

**关键词**：映射盘, 网络盘, subst, UNC, Vite, Failed to resolve entry, rollup, Could not resolve entry module, wrangler, d1, SQLITE_IOERR, SQLITE_IOERR_FSYNC, disk I/O error, cmd 批处理, 杮子乱码, 不是内部或外部命令, robocopy, MSYS_NO_PATHCONV, 端口占用, ZCode Write fsync

---

## 事故 1：Vite 在映射盘上连自己都解析不了

| 项 | 内容 |
|---|---|
| **症状** | `vite build` 报 `Failed to resolve entry for package "@tailwindcss/vite"`，换 `--configLoader runner` 后变成 `Failed to resolve entry for package "vite"`；`npm ls`/`node -e "import('@tailwindcss/vite')"` 全部正常 |
| **错误归因** | 一开始以为是 tailwind 版本 peer 依赖问题 / npm install-scripts 被拦 |
| **证据链** | ① `npm ls`、node ESM/CJS 导入被报包全部成功 ② `node -e "console.log(process.cwd()); console.log(fs.realpathSync.native('.'))"` → cwd=`Z:\...`，realpath=`\\V100\共享文件夹\...`，**不一致** ③ 换 UNC 路径作 cwd（两者一致）→ 配置加载通过了，但 rollup 报 `Could not resolve entry module "index.html"`（UNC 下 rollup 的 file URL 处理仍有坑） |
| **根因** | Vite 解析器用 `realpathSync.native` 结果做路径前缀判断；映射盘符 cwd 与 UNC realpath 错位导致所有包解析失败 |
| **为什么没早发现** | node/npm/tsc/esbuild/wrangler 在映射盘上都正常，只有 vite 的解析器有这个假设 |
| **修复** | 源码 robocopy 镜像到 `%LOCALAPPDATA%\gzt-build`（排除 node_modules），在本地 `npm install + vite build`，`dist` 再拷回映射盘。写成 `scripts\build.cmd` 一键完成，本地 node_modules 跨次复用 |
| **可复用教训** | **映射网络盘上做前端工程化：所有需要"真实路径一致性"的工具（Vite/Rollup）都走本地镜像目录，别在盘符路径上硬调** |

## 事故 2：wrangler d1 --local 在网络盘上 SQLite fsync 失败

| 项 | 内容 |
|---|---|
| **症状** | `wrangler d1 execute gzt --local --file=...` 报 `workerd/util/sqlite.c++: failed: SQLite failed; disk I/O error: SQLITE_IOERR (extended: SQLITE_IOERR_FSYNC)` |
| **证据链** | ① 同一命令在本地盘镜像目录一次成功 ② 项目 `.wrangler/state` 在 Z: 盘 ③ workerd 的 SQLite 要求真实 fsync，SMB 共享不支持 |
| **根因** | SQLite（workerd 内嵌）在网络共享上无法完成 fsync |
| **修复** | 同事故 1 的本地镜像：d1 execute / wrangler dev 全部从 `%LOCALAPPDATA%` 镜像目录跑。**远程操作（`--remote`、`wrangler deploy`）不受影响，可在映射盘直接跑** |
| **可复用教训** | **凡是本地起 SQLite/workerd 的工具链，状态目录必须在本地 NTFS 盘** |

## 事故 3：UTF-8 中文注释把 .cmd 批处理解析炸了

| 项 | 内容 |
|---|---|
| **症状** | 运行 build.cmd 输出 `'杮子乱码' 不是内部或外部命令`、`'/b' 不是内部或外部命令`，脚本行为完全错乱 |
| **错误归因** | 以为只是显示乱码 |
| **证据链** | ① 文件是 UTF-8 无 BOM + 中文注释 ② cmd.exe 按 OEM 码页（GBK/936）逐行解析批处理，UTF-8 多字节序列被切碎 ③ 部分行首字节恰好撞上特殊字符导致续行/命令拆分错乱 |
| **根因** | cmd 读 .cmd 不看 BOM，永远按系统 OEM 码页解析 |
| **修复** | 批处理**全文件纯 ASCII**（注释、echo 全部英文）；要中文输出就改用 PowerShell (.ps1 UTF-8 BOM) |
| **可复用教训** | **.cmd 文件里永远不写中文，一个字节都不要** |

## 事故 4（附带）：Git Bash 调 Windows 原生工具的坑两则

- **robocopy /MIR 被 MSYS 转成路径**：报 rc=16。解决：命令前加 `export MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*"`，或改用 `cmd //c "robocopy ..."`。
- **同端口残留多个 wrangler dev 监听**（SO_REUSEADDR 导致 netstat 能看到 2-3 个 PID，请求被旧实例接住，`--test-scheduled` 不生效）：`taskkill /F /PID` 单斜杠 + `MSYS_NO_PATHCONV=1`；按命令行精确清理用 PowerShell：`Get-CimInstance Win32_Process | ? { $_.CommandLine -match 'wrangler|workerd' } | % { Stop-Process -Id $_.ProcessId -Force }`。
- 另外：ZCode 的 Write 工具在该映射盘上报 `UNKNOWN: unknown error, fsync`，**是虚警，文件实际已写入**，写完用 `wc -c`/构建校验即可，别被吓退。

---

**验证状态**：真机实测通过（Windows 11 + Node 24 + wrangler 4.158 + vite 6.4；个人工作台项目全流程构建/本地 D1/部署脚本均用此方案跑通）。

> 关联：`29_GitHub访问 · ECS隧道三模式·PAC免管员分流·托盘exe`（同属"网络环境坑"）。
