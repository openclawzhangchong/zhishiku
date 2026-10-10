# 04 · frp 隧道 + Caddy 反代 + 域名公网暴露

> **摘要**：记录把本机 llama.cpp 推理服务经“阿里云 frps + Caddy 反代 + Cloudflare 域名”暴露到公网的完整链路与两个最反直觉的 Caddy 硬约束。结论来自 4 组对照实测，目标是对外暴露服务、绑域名、加 HTTPS、加鉴权时不再试错。
>
> **关键词**：frp；frps；Caddy；reverse_proxy；Cloudflare；jiuyeke.cn；HTTPS；auto_https；TLS；API 鉴权；远程管理；端口探测

---

## 一、踩过的坑（吃过哪些亏）

1. **Caddy 写 `auto_https off` 导致全 000**：它连**证书签发**一起关（端口在听但 curl 一律 000，Windows `SEC_E_ILLEGAL_MESSAGE`）。4 组对照实测才锁定正确写法。
2. **站点地址不写主机名签不出证书**：`https://:443` 同样签不出，必须 `https://jiuyeke.cn:443`。
3. **误判“80 被运营商封”**：临时 `TcpListener` 只 bind 没 accept，外部 curl 拿 RST → 差点改道；真实 Caddy 复测 80/443 完全正常。结论是“端口结论只能由真实服务给出”。
4. **误判“安全组全封”**：frps 没起时所有端口都 closed → 以为安全组封了；frps 起后 7000/8080 从公网直接可达。必须在服务已启动前提下扫端口。
5. **`curl -o` 直接覆盖在用的 49MB `caddy.exe` 写成 983KB 残废文件** → “指定的可执行文件不是此操作系统平台的有效应用程序”。
6. **CF SSL 模式误改成 Full (strict)** → 自签证书 526 失败；实测应为 **Full**。
7. **`https://域名:8080` 永远不通**：CF HTTP 端口组含 8080 但不做 TLS，HTTPS 组才做 TLS ⇒ 要修的是给源站补 443 反代。
8. **远程执行把自身 PowerShell 一起杀掉**：批量杀进程没排除自身（`$PID`）⇒ 执行命令的会话被杀。

## 二、怎样才能成功（成功做法）

- **链路拓扑**：`外网 → 阿里云 47.119.167.86（frps:7000 + Caddy）→[frp TLS]→ 家里 frpc → 127.0.0.1:8080`。
- **Caddy 正确写法（已锁定）**：`auto_https disable_redirects` + 站点地址带主机名（`https://jiuyeke.cn:443`）+ `tls internal`。成功判据：日志 `certificate obtained successfully, issuer=local`。
- **Caddy 反代必须保留流式**：`reverse_proxy 127.0.0.1:8080 { flush_interval -1 }`（删了 flush 流式变一次性蹦出）。
- **frps 用计划任务而非 sc.exe**：`frps.exe` 非服务程序，`sc.exe` 报 1053。`New-ScheduledTaskTrigger -AtStartup` + `Principal SYSTEM` + `RestartCount 3`。
- **鉴权最重要**：llama-server 默认零鉴权，隧道一开谁都能白用显卡 ⇒ 加 `--api-key`，并注入全部 8 个启动 bat + 局域网启动器。
- **远程管理主用 SSH（22 端口）**：阿里云 Windows 镜像自带 OpenSSH；`_ecs.py`（`-ps`/`--psf` 走 base64 + `-EncodedCommand` UTF-16LE，彻底规避引号与中文编码）。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **Caddy 两个硬约束（勿再试错）**：① 绝不能写 `auto_https off`；② 站点地址必须带主机名。`pki.ca.local "request is not supported"` 在 Windows 上无害。
- **CF 免费版把大陆流量调度到 LAX** ⇒ 首字节 1.8~3.7s（复用后 0.4~0.8s）；直连 ECS 仅 0.12s。提速＝灰云 + Caddy 自动 LE（代价：暴露源站 IP + 未备案风险）。
- **CF 段严重抖动**：分层探针域名经 CF 6/12 失败（统一 `curl exit=28` 25s 超时），直连 ECS/本机 12/12 ⇒ 故障在 Cloudflare 不在源站；边缘 IP `104.21.65.86` 已不通、`172.67.160.104` 尚可。
- **frpc 隧道反复重连**：`connect to local service ... actively refused` 是**本机服务没起**，非隧道故障。
- **下载一律先落 `.tmp`，校验字节数后再改名**；GitHub Release 国内 VPS 镜像测速：`ghfast.top` 520KB/s > `gh.ddlc.top` 462 > `gh-proxy.com` 64 > 直连 28 > `ghproxy.net` 16（且截断）。配 `--speed-limit 51200 --speed-time 10` 自动换源。
- **演练走旁路**：线上运行时用 `C:\_kit_test\` + 9443/9444 端口 + `t-` 前缀计划任务验证，收尾彻底删旁路目录。

## 四、具体怎么实现（实现方式）

**1. 服务端 frps（C:\frp\，v0.71.0）**
```toml
bindPort = 7000
auth.token = "<token>"
transport.tls.force = true
allowPorts = [{ 8080 }, { 8090 }]
```

**2. Caddy（C:\caddy\，v2.11.4）**：监听 80/8081(HTTP) + 443/8443(TLS)，全部 `reverse_proxy 127.0.0.1:8080 { flush_interval -1 }`。

**3. 域名 + Cloudflare**：NS 在 Cloudflare（橙云，源站 47.119.167.86）；SSL 模式 Full；线上全绿 `https://jiuyeke.cn/`、`/v1/*` 无密钥 401 / 带密钥 200；SSE 204 分块未缓冲。

**4. 鉴权**：`--api-key "sk-v100-..."`（存 `_frp/.apikey`），注入全部启动脚本；行为 `/health` 公开 200，`/v1/*` 无密钥 401。

**5. 远程通道**：`damoxing/_ecs.py`（`"cmd"` / `-ps "<脚本>"` / `--psf <本地ps1>` / `--put` / `--get`）；凭据来源 `ECS_PASS` > `secrets.local.json` > 交互；跑它必须用装了 paramiko 的 venv；批量杀进程 `Where-Object { $_.Id -ne $PID }`。

**6. 端口探测铁律**：判断端口是否被安全组拦，必须在服务已启动前提下扫；端口结论只能由真实服务给出（临时 TcpListener 只 bind 不 accept 会拿 RST）。

> 现成实现：`openclawzhangchong/aliyun-frp-domain-kit`（私有仓库，改 `config.json` 换新服务器）。
