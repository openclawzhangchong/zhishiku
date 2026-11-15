# 40 · frp 复用上线公网隧道：零新增入站端口 + 二改项目先跑冷启动

> **摘要**：把本机网关经 frp 发到公网域名，最常见的错误思路是新装一套 frps/开新端口。本篇验证了「复用存量 frps + Caddy 按域名反代 remotePort」的零新增暴露面做法，顺带沉淀三个独立坑：二改开源项目先跑"冷启动路径"（TDZ 崩溃）、ECS 拉 GitHub 单流被限速（aria2 多流秒解）、Git Bash 参数路径转换打穿 paramiko（MSYS_NO_PATHCONV）。

**关键词**：frp, frpc, frps, Caddy 反代, remotePort, allowPorts, 公网隧道, 反代网关, 管理面屏蔽, 边缘安全, Cannot access 'log' before initialization, TDZ, 暂时性死区, function 声明, const 箭头函数, 冷启动, aria2c -x16, GitHub 限速, 12KB/s, MSYS_NO_PATHCONV, Git Bash 路径转换, paramiko sftp ENOENT, 代理名冲突, frp 踢会话, 测试隔离, 核对门

---

## 事故/主题 1：想上公网隧道，但 ECS 已有 frps 和 Caddy，怕开新端口被扫

| 项 | 内容 |
|---|---|
| **症状** | 需要把本机服务发到公网域名；ECS 上 frps（别家隧道用）与 Caddy（80/443 已占）都在跑，不敢动配置也不敢开新入站端口 |
| **错误归因** | 以为必须改 frps 加 vhostHTTPPort 或新开端口，会牵连存量隧道 |
| **证据链** | ① 读 `/opt/frp/frps.toml`：`allowPorts=[8080-8090]`、无 vhostHTTPPort；② 看存量站点的 Caddyfile：`shuiyin → reverse_proxy 127.0.0.1:8081`，注释明写"frp 隧道→本地 WebUI"；③ 结论：存量模式就是 frpc 用 **TCP 代理占 808x remotePort**，Caddy 按域名反代它 |
| **根因** | 按 Host 路由的活 Caddy 已经在干，frps 只需要提供 TCP 隧道；两者配合即"每条隧道 = 一个 808x 端口 + 一段 Caddy 站点块" |
| **修复** | frpc（本机）：`type=tcp, localPort=8787, remotePort=8082` + `transport.tls.enable=true`（frps 强制 TLS）；Caddy 追加站点块反代 127.0.0.1:8082，`flush_interval -1` 保 SSE 流式不缓冲；DNS A 记录（Cloudflare API）；**frps 零改动、零重启、零新增入站端口**（8082 只有 Caddy 在 ECS 回环用） |
| **可复用教训** | 有存量 Caddy 的机器上配 frp 公网隧道：frpc 用 TCP 代理占 allowPorts 内的 remotePort，Caddy 加域名站点块反代之，frps 永远不用动 |

**验证状态**：真机实测通过（本机+独立出口双视角 /health 200；隔离测试 10 项 + 公网验收 9 项全过）

---

## 事故/主题 2：反代网关出了公网，管理接口全裸奔

| 项 | 内容 |
|---|---|
| **症状** | 网关的管理 API（密钥/账号/隧道开关）原本只设计给 127.0.0.1 面板用，隧道打通后公网也能访问 |
| **错误归因** | 以为 GATEWAY_KEY 只护 /v1/* 就够了 |
| **证据链** | ① 面板类管理端点在网关里不带鉴权（本机自用设计）；② 隧道是整端口 TCP 转发，公网等于拿到完整 8787；③ 在 Caddy 站点块加 `@block path /keys* /accounts* /frp* …` + `respond 403` 后，公网 /keys=403、本地不受影响 |
| **根因** | 反代把"本机自用"的管理面原样暴露给了公网 |
| **修复** | 边缘屏蔽：Caddy @block 管理路径，只放行 /health、/v1/*、/models；本机 127.0.0.1 直连不走边缘，不受影响 |
| **可复用教训** | 给只含"API+面板"的本机服务开公网隧道时，必须在边缘（Caddy/frps 侧）按路径屏蔽管理面，不能指望应用层鉴权 |

**验证状态**：真机实测通过（公网 6 个管理路径全 403，本地全可用）

---

## 事故/主题 3：二改开源项目，冷启动直接崩（TDZ + 空引用连环）

| 项 | 内容 |
|---|---|
| **症状** | `node zcgw-server.mjs` 启动即 `Cannot access 'log' before initialization`；修掉后第二个崩 `Cannot read properties of null (reading 'slice')` |
| **错误归因** | 以为是自己新加的代码引入的 |
| **证据链** | ① 崩点在模块顶层 `loadAccounts()`（347 行调用）→ 函数内 `log(...)`（263 行）；② `const log = ...` 定义在 660 行，ESM 按序求值，调用时还没初始化 → TDZ；③ 再修后崩在 `uid.slice`：本机 JWT 解析不出 `user_id`，`userIdFromJwt()` 返回 null |
| **根因** | 上游作者本地永远有 accounts.json，"缺数据文件"的冷启动分支从没被执行过；两个 bug 都藏在未覆盖路径里 |
| **修复** | ① `log` 改 function 声明（提升，全文件免疫 TDZ）；② label 判空兜底 `uid ? uid.slice(0,8) : 'anon'`（下游指纹派生本来就对空 uid 兜底） |
| **可复用教训** | 二改/克隆开源项目，动手前先在"空数据目录"跑一遍冷启动——作者跑得到的路径不等于你跑得到 |

**验证状态**：真机实测通过

---

## 事故/主题 4：ECS 从 GitHub 下载 release，单流 12KB/s；镜像全挂

| 项 | 内容 |
|---|---|
| **症状** | ECS `curl` 拉 frp release zip，120 秒只下了 1.4MB；ghproxy 系镜像全部 code:000；华为云镜像没有该版本（回 12KB 错误页） |
| **错误归因** | 以为 ECS 带宽就这么多 |
| **证据链** | ① 单流测速 ~12KB/s；② 换 `aria2c -x16 -s16 -k1M` 多 TCP 流直连 GitHub，13.9MB 秒级完成 |
| **根因** | GitHub CDN 对单连接限速，多流并发绕开；镜像站当前不可靠 |
| **修复** | `apt-get install -y aria2 && aria2c -x16 -s16 -k1M -d /tmp -o xxx.zip <github-release-url>`；下载完 sftp 拉回本机 |
| **可复用教训** | 国内 ECS 拉 GitHub release：先 aria2c -x16 多流直连，镜像站当备胎不当首选 |

**验证状态**：真机实测通过

---

## 事故/主题 5：Git Bash 给 paramiko 传 `/tmp/x` 参数，sftp 报 ENOENT

| 项 | 内容 |
|---|---|
| **症状** | `python ecs_run.py put f /tmp/zcodeapi.caddy` → `FileNotFoundError: [Errno 2] No such file`（远端明明有 /tmp） |
| **错误归因** | 以为是远端权限问题 |
| **证据链** | ① 同样的路径写在长命令串里（带引号）就正常，单独作为参数就崩；② Git Bash 的 MSYS 路径转换把独立参数 `/tmp/x` 改写成了 Windows 路径再传给 python |
| **根因** | MSYS2 对"看起来像 Unix 绝对路径"的独立参数做自动转换 |
| **修复** | 命令前缀 `MSYS_NO_PATHCONV=1`（或把路径放进带其他内容的引号串里） |
| **可复用教训** | Git Bash 下给 python/ssh/scp 传 Unix 绝对路径参数，统一加 `MSYS_NO_PATHCONV=1` |

**验证状态**：真机实测通过（加前缀后 put/get 正常）

---

## 事故/主题 6：测试 frpc 用同名代理，把线上隧道踢下线

| 项 | 内容 |
|---|---|
| **症状** | 想自动化测试"启用隧道"全链路，但测试实例的 frpc 代理名与线上一致 |
| **错误归因** | 以为 frps 会并存两个同名代理 |
| **证据链** | frps 对同名 proxy：新登录踢旧会话——测试一跑，线上隧道即断 |
| **根因** | frps 按 proxy name 唯一管理会话 |
| **修复** | 网关配置支持环境变量覆盖（`ZCGW_FRP_CONF`/`ZCGW_FRP_BIN`/`ZCGW_FRP_BINDING`），测试脚本从真实 frpc.toml 派生临时配置（代理名 zcodeapi-test、remotePort 8083），配合独立 `ZCGW_DATA_DIR` 做到核对状态也隔离 |
| **可复用教训** | 网关类服务加配置项时顺手留环境变量覆盖入口，是测试隔离的最低成本方案；frp 同名代理互踢，测试必换名 |

**验证状态**：真机实测通过（隔离实例 10 项测试期间线上隧道 /health 持续 200）

---

## 可复用教训速查

- frp 公网隧道上 Caddy 存量机：TCP 代理占 808x + Caddy 按域名反代，frps 不动
- 管理面上公网必须在边缘按路径屏蔽
- 二改开源项目先空数据冷启动跑一遍
- ECS 拉 GitHub release 用 aria2c -x16
- Git Bash 传 Unix 路径参数加 MSYS_NO_PATHCONV=1
- frp 测试代理必须换名（同名互踢）

> 关联：`29_GitHub访问`（隧道 push 的通用做法）、`38_配额验证`（限流/防刷思路）。
