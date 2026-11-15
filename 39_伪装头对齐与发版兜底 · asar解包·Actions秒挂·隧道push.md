# 39 · 伪装头对齐与发版兜底：asar解包 · Actions秒挂 · 隧道push · API直传

> **摘要**：反代/机器人伪装官方客户端时，抄来的版本常量必然过期——真版本号和出站头要从官方安装目录解包提取，`app.asar` 字符串明文可直接 grep，不用完整解包。配套两个发版坑：GitHub Actions **秒挂（0 step）** 是账单封锁不是代码问题；commit 还在本地时 `gh api PATCH refs` 会 422 "Object does not exist"——API 只会移指针不会传对象，必须走真 push（ECS 隧道兜底）。

**关键词**：asar, Electron, 伪装, masquerade, User-Agent, X-IDE-Version, version常量, Actions, 秒挂, 0 step, billing, spending limit, recent account payments, Object does not exist, 422, gh api, refs, ECS隧道, curloptResolve, NSIS, 本地打包兜底

---

## 事故/主题 1：伪装版本常量过期 —— "上游会不会查"不能靠猜

| 项 | 内容 |
|---|---|
| **症状** | 反代伪装的官方客户端版本停在 5.5.4/CLI 2.137.1（抄上游仓库的），用户机器官方客户端已升到 5.11.3/CLI 2.176.0；且逐头比对后发现 5 处偏差：缺 `X-IDE-Version`/`X-Private-Data`/`X-Agent-Intent`，`X-IDE-Type` 发 `desktop`（真机发 `WorkBuddy`），`X-Product` 发 `WorkBuddy`（真机发 `SaaS`） |
| **错误归因** | 最初以为"看看别的开源仓库怎么写的"就能拿到新值——结果上游自己也停在旧值，别人仓库没有增量价值 |
| **证据链** | ① `(Get-Item WorkBuddy.exe).VersionInfo.FileVersion` = 5.11.3（新版装 `C:\Program Files\`，旧 AppData 路径只剩空壳）；② `app.asar.unpacked/cli/package.json` 的 `@genie/agent-cli` version = 2.176.0；③ `grep -aoE` 直接在 asar 里搜出 UA 拼装模板与身份头构造（字符串明文存储）；④ 真机日志 `~/.workbuddy/logs/` 里整条 UA 与 asar 解析互相印证 |
| **根因** | 伪装常量是"抄来的静态快照"，官方客户端升级后必然漂移；且不同请求路径的头集合不一样，只对齐 UA 不对齐全组身份头等于半伪装 |
| **为什么没早发现** | 反代一直 200 能用，没有报错逼你查；直到用户主动升级官方客户端才暴露时间差 |
| **修复** | 以解包为准改常量 + 补齐/修正 5 处头（详见本仓库 docs/feature-masquerade-alignment.md）；`cargo check` + 341 测试 + 真机流式冒烟（200/SSE/无 11128）后才发版 |
| **可复用教训** | 伪装客户端身份：版本号读 exe 的 FileVersion、CLI 版本读其 package.json、出站头构造 `grep -aoE "模式" app.asar`（明文可搜），全部以真机解包为准，别抄任何仓库的常量 |

**验证状态**：真机实测通过（流式对话 200、SSE 正常、无 11128）

## 事故/主题 2：GitHub Actions 3-4 秒挂、0 个 step —— 账单封锁，不是代码

| 项 | 内容 |
|---|---|
| **症状** | push main / 打 tag 后三条流水线（test/build/desktop）全部 completed:failure，每个 job 0 个 step，耗时 3-4 秒 |
| **错误归因** | 以为 tag 打错、或 workflow 文件坏了 |
| **证据链** | ① `gh run view <id> --json jobs` → `steps: 0`；② `gh api .../check-runs/<job_id>/annotations` → "The job was not started because recent account payments have failed or your spending limit needs to be increased"；③ 与两个月前 v3.2.0 发版卡死时的 annotation 一字不差——账号付款又失败/额度又超了 |
| **根因** | GitHub 账号级 Billing 问题把整个 Actions 停了（public 仓库也照停），run 在启动阶段就被拒，跟代码零关系 |
| **为什么没早发现** | 上次解封后一直绿，惯性以为不会再犯；付款方式问题会复发 |
| **修复** | 短期：本地 NSIS 打包兜底（`npx tauri build --bundles nsis`），产物 `gh release create vX.Y.Z --verify-tag <exe>` 手工发版——build.yml 的 create-draft 步骤带 `\|\| "already exists, continue"` 容错、publish 步骤只做 `--draft=false --latest`，后续 CI 重跑不会和手工 Release 打架；长期：只有账号主人能去 Settings → Billing & plans 修付款/额度，修完 `gh run rerun <id>` |
| **可复用教训** | CI 秒挂先看两个指标：**job steps=0 + annotation 含 "account payments"** 即账单封锁，别排查代码；修复只有账号主人能做，Agent 能做的是本地打包 + `gh release create` 兜底闭环（先确认 workflow 对已存在 Release 有容错再动手） |

**验证状态**：真机实测通过（本地 NSIS 包 + 手工 Release 发出；CI 重跑容错逻辑从 build.yml 源码确认）

## 事故/主题 3：commit 在本地时 `gh api PATCH refs` 报 422 "Object does not exist"

| 项 | 内容 |
|---|---|
| **症状** | git push 连不上（github.com:443 TCP 超时），改用 `gh api -X PATCH .../git/refs/heads/main -f sha=<本地commit>` 想直接推指针，报 422 `Object does not exist` |
| **错误归因** | 以为 gh api 通道坏了或 sha 写错 |
| **证据链** | ① 本地 `git rev-parse HEAD` 与 sha 一致；② `gh api .../commits/main` 能访问（api.github.com 不被墙）；③ 缺的是：这个 commit 对象**只存在于本地**——git push 会传输对象，API 移指针只认 GitHub 上已有的对象 |
| **根因** | `PATCH/POST refs` 是指针操作，不传输对象；commit 没上云之前指针无处可指 |
| **为什么没早发现** | "gh api 能替代 git push"这个记忆太粗——它只在对象已经在 GitHub 上时成立（比如补打指向远端已有 commit 的 tag） |
| **修复** | 走真 push 传对象：ECS SSH 隧道 forward 模式（`python gh_tunnel.py forward 900 443` 起隧道 → curl `--resolve github.com:443:127.0.0.1` 验证 200 → `git -c http.curloptResolve="github.com:443:127.0.0.1" push`），两次一把过 |
| **可复用教训** | `gh api` 改 ref 的前提是**对象已在 GitHub**；commit 只在本地时老老实实走隧道 push——forward 模式管 git、`-c http.curloptResolve` 只影响当条命令不改全局 |

**验证状态**：真机实测通过（main + tag 隧道推送成功，流水线随即被触发）

## 事故/主题 4：ECS 隧道也挂 —— API 对象直传是 push 的第三条路

| 项 | 内容 |
|---|---|
| **症状** | 直连 github.com:443 挂（老问题），这次连 ECS 隧道也不行了——SSH 22 能连但转发请求全 000，进 ECS 本机实测 `curl https://github.com` = 000（ECS→github 出站故障） |
| **错误归因** | 反复重启隧道、怀疑脚本坏了 |
| **证据链** | ① ECS 22 端口 Test-NetConnection 通；② 隧道进程活着但转发 000；③ ECS 本机 curl github.com 000 → 结论是 ECS 侧出站坏，不是本地隧道问题 |
| **根因** | 三条路里只剩 api.github.com 通道还活着（gh 一直能用）——git 的对象传输其实可以用 REST API 手工完成 |
| **修复** | blob→tree→commit→refs 逐层经 REST 建：`git ls-tree` 拿每个文件的 mode+sha → `git cat-file blob` 读内容 base64 → POST `/git/blobs` → POST `/git/trees`（base_tree=父提交的树）→ POST `/git/commits` → PATCH `refs/heads/main`。已泛化成工具（workbuddy-switch2api 仓库 `scripts/api_push.py`，任意仓库 `--git <路径> --repo <owner/name> --tag vX.Y.Z`） |
| **可复用教训** | push 的第三条路 = API 对象直传：blob/tree sha 由内容决定校验必过；commit sha 想与本地零分叉见下面三个坑；一次只推一个 commit（parent 必须等于远端 main），多个就改一次推一次 |

**验证状态**：真机实测通过（main+tag 上云且 sha 与本地完全一致；工具 dogfood 推送自己成功）

### API 直传的三个 sha 坑（不踩就零分叉）

1. **GitHub 建 commit 会去掉 message 结尾换行**——本地提交用
   `printf '%s' "msg" > f && git commit -F f --cleanup=verbatim`（msg 不带结尾换行），
   或推完让工具自动按远端 message 重建本地 commit；
2. **远端 JSON 的 date 被规范成 UTC，不能拿来重建本地 commit**——复刻本地
   `git cat-file commit HEAD` 里的 author/committer **原始行**（含 ±HHMM 偏移）才对；
3. **`gh api` 不支持 `-R`**（端点写全路径 `repos/{owner}/{repo}/...`）；
   另外 `gh api POST /releases -f tag_name=...` 会**自动打 tag** 指向当时的 main，
   建 Release 后要核对 tag 指向，挪 tag 用 `PATCH refs/tags/<name>`（tag 已存在时
   POST 报 422 "Reference already exists"）。

> 关联：`29_GitHub访问 · 隧道`（forward/socks5 细节）、`02_发版流程 · tag`。
