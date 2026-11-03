---
id: 29
title: GitHub访问
stack: ["git", "tunnel"]
keywords: ["ECS隧道", "PAC分流", "托盘exe"]
symptoms: ["push被墙", "打不开GitHub"]
whenToRead: GitHub 被墙
oneLiner: ECS SSH 中转三模式；上传必死下载偶通是上行干扰
---

# 29 · GitHub 访问：ECS 隧道三模式（git / 浏览器 / PAC 分流托盘 exe）

> **摘要**：本机直连 github.com 时**下载偶通、上传必死**（`git push` 连续 10+ 次全灭：21 秒连接超时 / `send-pack: unexpected disconnect` 中途 RST）。改用阿里云 ECS 做 SSH 中转后，push / clone / 32MB Release 二进制上传**全部一把过**，且 ECS→GitHub 实测 0.27s。沉淀为独立工具仓库（forward / socks5 / 托盘 exe 三模式），本篇是原理、命令与踩坑全记录。
>
> **关键词**：SSH 隧道、SOCKS5、PAC 分流、AutoConfigURL、curloptResolve、免管理员、托盘常驻、PyInstaller、paramiko、CLOSE_WAIT 假死
>
> **来源**：zcmusic（algermusicplayer）私库 push 事故 2026-10-30 → 工具化 `openclawzhangchong/ecs-github-tunnel`（私库，Release 提供成品 exe）
>
> **关联**：03 篇（API 四步法，无 SSH 条件时的替代）；10 篇（Pages 25MiB 等 CF 平台约束）

---

## 一、判据：是"被墙"还是"网断了"

- **下载（fetch/clone）偶通、上传（push）必死**——别被 fetch 成功迷惑，这是典型的上行干扰特征。
- 换多个 GitHub IP（hosts 固定）结果一致 → 不是单 IP 问题。
- 改 hosts 需 UAC 弹窗且固定 IP 会过期失效，**不要走这条路**，用下文的 `http.curloptResolve`（免管理员、不污染全局配置）。
- 中转机判据：ECS 上 `curl -sI https://github.com` 应为 200（实测 0.27s）；若 ECS 也不通，本篇方法失效，转 03 篇 API 四步法。

## 二、三种模式（按需选）

### 模式 1：forward —— git push/pull（最常用）

```bash
python gh_tunnel.py forward 900 443          # 本地 443 → ECS → github.com:443
git -c http.curloptResolve="github.com:443:127.0.0.1" push origin main
```

- Windows **非管理员可绑 443**；SNI/Host/证书校验完全不受影响（纯 TCP 转发）。
- `-c` 只影响当条命令，不改全局配置；可配仓库级 alias 固化。

### 模式 2：socks5 —— 浏览器打开 GitHub 网页

```bash
python gh_tunnel.py socks5 900 1080          # 默认只放行 GitHub 系域名，--any 全放行
# 浏览器：msedge.exe --proxy-server="socks5://127.0.0.1:1080" 或 SwitchyOmega
```

- 验证：`curl --socks5-hostname 127.0.0.1:1080 https://github.com` → 200。
- SOCKS5 把域名交给代理端解析，**DNS 污染问题一并解决**。

### 模式 3：托盘 exe —— PAC 分流常驻（日常推荐）

`GitHubEcsTunnel.exe` 双击进托盘，核心是 **PAC 分流**：GitHub 系域名 → SOCKS5 走 ECS，其余 → DIRECT 本机直连。

| 环节 | 做法 | 为什么 |
|---|---|---|
| 系统代理 | HKCU 写 `AutoConfigURL = http://127.0.0.1:1081/pac` | **免管理员**；Edge/Chrome 全认 |
| 即时生效 | 广播 `WM_SETTINGCHANGE "Internet Settings"` | 浏览器不用重启 |
| 退出还原 | 启动时先存旧值（AutoConfigURL/ProxyEnable/ProxyServer），退出写回 | 不留残留 |
| 开机自启 | HKCU `...\CurrentVersion\Run` 加值 | 免管理员的自启位 |
| 单实例 | 命名互斥体 `CreateMutexW`，`GetLastError()==183` 即退出 | 重复双击不炸 |
| 打包 | `pyinstaller --onefile --noconsole`，pystray+PIL 画图标 | 32MB 单文件 |

实测：`github.com` 真实加载（走 ECS）✅；`baidu.com` 真实加载（直连）✅——**分流双向都对**。
注意：git 命令行不吃系统代理，push 仍走模式 1。

## 三、从 ECS 实测的域名覆盖面（2026-10-30）

| 域名 | 状态 |
|---|---|
| github.com / api.github.com / codeload / objects / avatars / raw | ✅ 全通 |
| **gist.github.com** | ❌ 超时——**阿里云到该边缘 IP 路由不通**，环境限制，别浪费时间去排查脚本 |

## 四、踩坑清单（全是当天真踩的）

| 坑 | 现象 | 解法 |
|---|---|---|
| SSH transport 假死 | 跑一阵后全部 000，端口还在 LISTENING 但连接进 CLOSE_WAIT | `ensure_transport()` 自动重连 + 通道级一次重试；还不行重启进程 |
| schannel 重协商循环 | Windows curl 对 GitHub TLS 反复 renegotiation，首请求偶慢 | 重试即好；浏览器（Chromium 内核）无此问题 |
| 强杀进程不还原代理 | `taskkill //F` 跳过退出钩子，AutoConfigURL 残留指向死 PAC | 重新双击 exe 再点「退出并还原」；浏览器对拉不到的 PAC 会自动回落 DIRECT，属可接受降级 |
| Python `global` 位置 | `global transport` 写在使用之后 → SyntaxError | global 声明必须在函数体最前 |
| Git Bash 路径转换 | curl `-w "/xxx -> %{redirect_url}"` 输出变成 `C:/Program Files/Git/xxx` | `export MSYS_NO_PATHCONV=1` 再诊断 |
| 贪快走 hosts | 固定 IP 过期、还要 UAC | 全程 curloptResolve / SOCKS5，不碰 hosts |

## 五、安全铁律（代理工具的生命线）

1. **ECS 侧零新增监听端口**——一切流量复用 SSH 22；裸 SOCKS 服务端公网可达时几小时内会被扫爆，还可能被云厂商封号。
2. **本地只监听 127.0.0.1**，绝不 0.0.0.0。
3. **socks5/PAC 默认域名白名单**，防止浏览器全量流量误走小水管 ECS（带宽通常 1~5Mbps，git 操作够用）。
4. 凭据只存本地 env 文件（`ECS_HOST`/`ECS_USER`/`ECS_PASS`），脚本与文档一律不落明文。
5. 用完即关或设存活秒数（脚本默认自动退出）。

## 六、与 03 篇的分工

- **03（API 四步法）**：机器上没有可用的 SSH 中转机时，纯靠 `api.github.com` 拼 blob/tree/commit/ref。
- **29（本篇，SSH 隧道）**：有一台国际路由干净的 ECS 时，改动小、保语义（真正的 git push），日常首选。
- 两者判据先行：先探 `api.github.com` 与 SSH 中转机谁通，别死磕一条路。
