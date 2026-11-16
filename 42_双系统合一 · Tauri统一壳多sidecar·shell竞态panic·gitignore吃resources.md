# 42 · 双系统集成合一：Tauri 统一壳 + 多网关 sidecar（api-hub）

> **摘要**：把两个技术栈完全不同的现成产品（zcode-hub：Tauri 壳 + PyInstaller FastAPI sidecar；workbuddy：Tauri 壳 + Rust axum 单文件）合并为一个新 App，正解是「**统一壳托管多网关 sidecar**」——保留两个网关本体的全部功能，只重做壳层（托盘双入口/门户页/统一进程托管），而不是重写任何网关。本篇记录架构决策、四个实测坑（shell 插件竞态 panic、gitignore 吃掉 NSIS 资源、空仓库 API 推送限制、端口冲突探活）。

**关键词**：api-hub, Tauri 统一壳, 多 sidecar, 双网关, 托管子进程, CREATE_NO_WINDOW, externalBin, resources 目录, frontendDist 绝对路径, shell 插件竞态, state called before manage, 空仓库 API 推送 409, Contents API bootstrap, NSIS 遵循 gitignore, 端口冲突 7863/7865, 免密护栏, 环境污染

---

## 架构决策：合并 ≠ 重写

| 项 | 内容 |
|---|---|
| **背景** | 两个现成产品：zcode-hub（Tauri 壳 + PyInstaller FastAPI sidecar :3000，ZCode/Z.AI 池）与 workbuddy-switch2api（Tauri 壳 + Rust axum 单文件 :7863，WorkBuddy/CodeBuddy 池，免费优先调度）。用户要求新建 api-hub 合二为一，原产品不动 |
| **错误路线** | 重写统一网关支持所有 provider——workbuddy 的伪装对齐/免费优先调度逻辑复杂，数周工作量且高风险 |
| **正确路线** | **统一壳托管多网关 sidecar**：新 Tauri 壳复制自 zcode-hub 壳（已验证），托盘菜单双入口，三个子进程（zcode backend :3000 + wb-switch :7865 + frpc）统一 spawn/kill，门户页双卡片状态灯 |
| **可复用教训** | 合并两个「壳+服务」型产品 = 只做壳层整合，网关本体作为黑盒 sidecar 原样打包；数据目录各归各（AppData apihub / home .workbuddy） |

**验证状态**：真机实测通过（四进程共存、双后台 200、公网 200）

---

## 坑 1：Tauri setup 线程里 `app.shell()` → panic（state called before manage）

| 项 | 内容 |
|---|---|
| **症状** | 壳进程活着但子进程一个都没拉起；开发版从 bash 跑立刻 panic：`state() called before manage() for tauri_plugin_shell::Shell` |
| **根因** | app setup 派生的线程与 **shell 插件的状态 manage 存在竞态**——线程第一时间调 `app.shell()`（ShellExt）时插件 state 尚未注册 |
| **修复** | 不依赖 tauri-plugin-shell 的 sidecar 机制：全部子进程改 `std::process::Command` + `CREATE_NO_WINDOW`，kill 句柄用 `Mutex<Vec<Child>>` 自托管；Cargo.toml 移除 shell 插件（连竞态源一起消灭） |
| **可复用教训** | Tauri v2 里从 setup 派生的线程调用 `app.shell()`/其他插件 Ext 存在初始化竞态——要么在主线程先 `app.state::<T>()` 预热再传句柄，要么干脆不用插件、走 std + CREATE_NO_WINDOW |

**验证状态**：真机实测通过（改后四进程稳定共存，多次重启无 panic）

---

## 坑 2：NSIS 打包遵循 .gitignore → resources 目录缺文件 → 后端启动即崩

| 项 | 内容 |
|---|---|
| **症状** | 安装后 `frontend/`、`captcha_node/` 整个目录缺失（6669 文件只剩壳 exe），后端启动即崩（statics RuntimeError） |
| **根因** | **tauri 打包器收集 resources 时遵循 .gitignore**——而我把 frontend/captcha_node/binaries 加进了 .gitignore（本意是防 28MB exe 入库），把要打包的资源也排除了 |
| **修复** | .gitignore 精确到文件（只忽略 exe/toml），目录本身不 ignore；打包前断言 resources 关键文件存在 |
| **可复用教训** | Tauri resources 会吃掉 .gitignore 内的文件——ignore 规则必须逐文件写，不能整目录 ignore 要打包的资源 |

**验证状态**：真机实测通过（修复后安装包 6669 文件齐全）

---

## 坑 3：空仓库的 GitHub API 推送限制（blobs 409 / contents 404）

| 项 | 内容 |
|---|---|
| **症状** | gh repo create 新建（空）仓库后：git/blobs 返回 409 "Git Repository is empty"；contents PUT 返回 404 |
| **根因** | GitHub 限制：空仓库上 Git Data API 与 Contents API 均不可用，必须先有第一个提交 |
| **修复** | 两段式：① Contents API PUT 一个 README（bootstrap 首提交，空仓库唯一可用的写入端点）② 拿到首提交 sha 后，blobs→tree→commit→ref 补全其余文件 |
| **可复用教训** | 空仓库只能用 Contents API 引导首提交，之后 Git Data API 全通 |

**验证状态**：真机实测通过（97 文件补全推送成功）

---

## 坑 4：端口冲突探活——复用用户已有账号库时避开原产品端口

| 项 | 内容 |
|---|---|
| **症状** | wb-switch 在 api-hub 里启动后反代入口 bind 失败（0.0.0.0:7863 被占）——用户正在跑 workbuddy 桌面版 |
| **根因** | workbuddy 桌面版与 wb-switch-server 默认同端口 7863；账号库共用 home 下 .workbuddy |
| **修复** | api-hub 的 wb 实例用 `WB_SWITCH_PORT=7865` + `--no-open`；账号库沿用 .workbuddy（复用用户已有账号池是 feature） |
| **可复用教训** | 集成第三产品时先探它的默认端口占用，错位分配；共库是卖点要保住 |

**验证状态**：真机实测通过

---

## 补充：本轮再次实证的旧坑

- `>nul` 在 Git Bash 创建保留名文件 → `git add -A` 卡死（PowerShell `\\?\` 前缀删）——本轮 delete 后 git 索引仍缓存，需 `git add --ignore-errors -A`
- `tasklist //FI` 间歇误报 → PowerShell Get-Process 复核
- `move /y` 是 cmd 命令，Git Bash 里静默失败 → 用 `mv`
- PyInstaller --noconsole 隐藏启动必须重定向 stdout 到文件（无 stdout 卡死不 bind）

> 关联：`40_frp复用上线公网隧道`、`41_单文件安装包打包`、`29_GitHub访问`。
