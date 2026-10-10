# 06 · GitHub 仓库管理与网络封锁下推送

> **摘要**：记录本机在 `github.com:443` 被 reset（CONNECT 502）导致 `git push` 必失败的环境下，如何用 Git Data API 逐对象推送完整提交历史、如何把公开仓库批量转私有、以及 PAT 安全清单。目标是网络受限时仍能可靠地把仓库推上去、改可见性，且不留明文凭据。
>
> **关键词**：Git Data API；GitHub；网络封锁；PAT；公开转私有；增量推送；blobs；trees；commits；refs；MCP 限制；凭据安全

---

## 一、踩过的坑（吃过哪些亏）

1. **`git push` 必失败**：本机代理对 `api.github.com`/`codeload` 正常，但 `github.com:443` 被 reset（直连 20.205.243.166 也超时）→ git 协议不可用。误以为换镜像就能 push，实则 `gh-proxy.com` 等只用于**下载**不能 push。
2. **新仓库空时 `/git/blobs` 返回 409 "Git Repository is empty"**：一开始直接走 Git Data API 报错，必须先用 `PUT /contents/<file>` 提交一个文件完成初始化。
3. **`POST /git/commits` 的 `parents` 格式错** → 422 误导信息（Ruby 风格误导，items 并非 parents 字段名）。首个 commit 无父一路通过，只有推第二个（有父）commit 才炸。
4. **靠 SHA 判“是否已推送”**：本地 SHA 与 GitHub 重建 SHA 必然不同（tree SHA 也不同）→ 误判重复或遗漏。改用 `(tree_sha, message.strip())` 做 key 比对。
5. **8 线程并发上传触发 403 secondary rate limit** → 一度以为是 token 权限问题，实为限流。
6. **公共 fork 转私有全 422 失败**：`{"message":"Public forks can't be made private"}`；API 不允许把公开 fork 直接转私有，必须先 detach（只有 Web UI 能做）。
7. **长中文提交信息写在命令行触发安全钩子被拒**（"Invoking PowerShell from Bash"）→ 命令行带 token 还会触发审批超时无法执行。

## 二、怎样才能成功（成功做法）

- **Git Data API 逐对象推送（绕过封锁）**：`blobs → trees → commits → refs`，保留完整 commit message/author/date，无需 git 协议。工具 `push_via_api.py` / `_gh.py`。
- **新仓库初始化**：先 `PUT /contents/<file>` 提交一个文件，再走 Git Data API。
- **`parents` 用 SHA 字符串扁平数组**，不是对象数组。
- **远程已存在 commit 识别**：用 `(tree_sha, message.strip())` 比对；`/commits` 列表不返回 tree，必须逐个 `GET /git/commits/{sha}`。
- **串行 + `sleep 0.25`**（约 4 req/s）即稳定通过 403 限流（不要用 8 并发）。
- **refs/tags 幂等**：标签已存在时 `POST /git/refs` 报 422 → fallback 到 `PATCH /git/refs/tags/{name}` + force。
- **公开转私有走 REST API + classic PAT**：`GET /user/repos?type=owner` 分页拉全 → 过滤 `private==false` → `PATCH /repos/{full_name}` `{"private":true}`。fork 转私有必须走 Web UI（Settings → Change visibility → Make private，UI 会自动 detach）。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **GitHub MCP server 无创建仓库权限**（403 Resource not accessible by integration）→ 建仓库/改可见性必须用 PAT，不能用 MCP。
- **MCP 没有“改可见性”工具**（只有 create_or_update_file / PR / issue / repos 读）→ 批量转私有只能 REST API。
- **token 上命令行会被安全钩子拦** → 令牌从环境变量 `PAT` 或同目录 `.tok` 读取；提交信息从文件读 `_gh.py push --message @_msg.txt`。
- **中断重跑**：blob/tree 缓存（`.push_cache.json`/`.tree_cache.json`）避免重复上传，支持增量推送。
- **fork 转私有 24 个全 422**：只有非 fork 的公开仓库能被 REST 自动化覆盖。

## 四、具体怎么实现（实现方式）

**1. Git Data API 推送关键坑速查**
- 新仓库空 → 先 `PUT /contents/<file>` 初始化，再走 blobs/trees/commits/refs。
- `parents` = SHA 字符串数组，不是对象数组。
- 8 并发 → 403 → 串行 + `sleep 0.25`。
- 标签已存在 → `PATCH /git/refs/tags/{name}` + force。
- PAT 从 `.tok`/环境变量读，别上命令行。

**2. 命令文本安全钩子**：长中文提交信息从文件读；一切推送走 Git Data API；镜像可用 `ghfast.top` 等（仅下载）。

**3. 安全清单（每次推送后必做）**
- 删 `.tok` / 会话内明文 PAT → 去 https://github.com/settings/tokens **吊销**。
- 仓库 `DEPLOY.md`/`scripts` 里硬编码的 AUTH_TOKEN / 密码 → 改占位符或环境变量强制传入。
- ECS 明文密码（`_ecs.py` / `secrets.local.json`）→ 注意保管，不进仓库。

> 账号 `openclawzhangchong`：PAT 存工作区 `.tok`（用完必须删除 + 去控制台吊销）。可用镜像 `ghfast.top` / `gh-proxy.com` / `ghproxy.net` 全 200（仅下载）。
