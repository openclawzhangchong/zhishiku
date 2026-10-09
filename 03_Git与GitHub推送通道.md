# 03 · Git 与 GitHub 推送通道

> **摘要**：本机 `git push` 到 github.com 常年 502 / Connection reset，逼出一套"Git Data REST API 四步法"推送方案，也踩烂了 blob 编码、CRLF、树校验、stash 毁库等一堆坑。本篇是可直接照抄的推送脚本要点与事故清单。
>
> **关键词**：API 四步法、blob/tree/commit/ref、base_tree、CRLF、git stash 毁库、CI 日志下载、PAT、单一通道
>
> **来源**：xiaoleapp、xiaole-app-xcode-2027、jiaoxue、xinxibook、youxi、编程调试

---

## 一、为什么走 API 四步法

- 本机到 `github.com:443` 直连不通（502 / `schannel: server closed abruptly` / HTTP2 framing error / `Connection reset by 20.205.243.160`），但 **`api.github.com` 通**。
- 判据：**github.com 不可达 ≠ 网络全断**，先探 `api.github.com`。
- 配套：`env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY` + curl `--noproxy "*"`。

---

## 二、四步法要点（脚本可直接照抄）

```
① 建 blob   POST /repos/{o}/{r}/git/blobs   {'content': base64, 'encoding':'base64'}
② 建 tree   POST /repos/{o}/{r}/git/trees   base_tree = 远端完整 tree
③ 建 commit POST /repos/{o}/{r}/git/commits parents = 远端 HEAD
④ 更新 ref  PATCH /repos/{o}/{r}/git/refs/heads/{branch}  {'sha':<40位>, 'force':true}
```

### 每一步的硬约束（全是踩出来的）

| 环节 | 铁律 | 踩过的坑 |
|---|---|---|
| 内容取字节 | 用 `git cat-file blob <sha>`（或 `git rev-parse HEAD:<f>`），**不要读磁盘文件** | CRLF 让磁盘字节 ≠ 提交对象字节，blob sha 对不上 |
| blob 编码 | `base64.b64encode`，**不能**用 `raw.hex()` + `encoding:base64` | blob 损坏、tree sha 不匹配 |
| blob 校验 | 建完校验 `b['sha'] == want`，不等立即中止 | 静默损坏推上去才发现 |
| base_tree | 用 `GET /git/trees/{branch}?recursive=1` 拿**完整远端 tree**，只替换改动项 | 只提交改动文件 → 仓库其它文件全丢 |
| 删除文件 | tree entry 写 `{"sha": null}` | `git diff --name-only` **不含删除项** → 必须 `--name-status --no-renames` |
| 中文路径 | `git -c core.quotepath=false`；Git Bash 下 `git add "中文.md"` 报 pathspec 不匹配 → 用 `git add -u` | 编码问题 |
| parents | 必须用**远端 HEAD**（API 查，不手拼） | 用本地 sha → 平行历史 |
| PATCH ref | sha **必须完整 40 位**（12 位报 "At least 40 characters"）；新链不同源必须 `force:true` | 422 |
| API 前缀 | 全路径 `/repos/{owner}/{repo}/git/...`，不能写裸 `/git/blobs` | 404 |
| 推送后 | 抽样 8 个文件**字节回读**比对 | 只读 sha 不够保险 |

### 一致性证明
- **tree sha 相等 = 内容等价**（git 内容寻址的数学保证）。本地与远端"同树异 SHA"是正常的——GitHub 对 API 提交附加自家 gpgsig 签名头，实体 API 不返回，无法逐字节重建。**放弃 SHA 对齐，以 tree sha 为准**。
- 不要用**字节数**比对判定一致（CRLF/LF 差 258 字节会假报警），要比 git blob sha。

---

## 三、仓库与分支

- **仓库名混淆是真实事故**：`xiaole-web-2027`（带连字符）是 size=0 空仓库，真正有内容的是 **`xiaole-web2027`**。推送前先 `GET /repos/{o}/{r}/branches` 确认非空。
- 新仓库默认分支是 **main**（老 Flutter 仓库是 master），别想当然。
- 删除重建同名仓库有 **422 名称保留期**（实测 2 分钟+）。
- 空仓库：`/git/blobs` 返回 409 → 先 `PUT /repos/{o}/{r}/contents/README.md` bootstrap，或建仓库时 `auto_init=true`。
- 建 tag 报 422 "Object does not exist"：**sha 必须现查**（`GET /repos/<repo>/git/ref/heads/main`），绝不手拼或截断。
- **同一仓库自始至终只用一种推送通道**：本地 git push 与 API 四步法混用 → 产生 main/master 平行历史，仓库首页显示旧代码。

---

## 四、🔴 毁库事故：`git stash` 做对照实验

**过程**：`git stash push -u && flutter test ... ; git stash pop` 被 SIGTERM 中断 → `.git/refs` 目录消失、objects 只剩 43 个、commit 全取不到。

**恢复步骤**：
1. 先备份工作区（`cp -r lib/ pubspec.yaml README.md` 到安全目录）
2. `mv .git .git_damaged_bak`
3. `git init`
4. **精确 `git add`**（不要 `git add -A`）
5. 推送改 base_tree 模式 + blob sha1 前检 + 推后逐文件字节回读

**Windows 下禁用 `git stash` 做对照实验**，改用 `cp` 备份到仓库外。

**第二个坑**：重建仓库后 `git reset --hard` 导致文件**静默丢失**——因为重建时只 add 了部分目录，`reset --hard` 只恢复"HEAD 有变化"的文件。根治：**仓库初始化后第一次提交必须纳入完整项目树**（含 android/ios/test/web/.github/docs，实测 196 文件）。补救：`git checkout <commit> -- .` 从树强制恢复。

---

## 五、CI 与产物

- workflow 建 Release 必须配 `permissions: contents: write`（否则 403）。
- 通过 contents API 推含换行的内容：**禁止在 heredoc 手写 `\n`**（会被写成字面量 `/n`，YAML 解析失败）→ 用 `chr(10)` 拼接或整体替换文件。
- **日志与产物下载两段式**：带鉴权拿 302 Location → **第二段不带任何鉴权头**（带 Authorization 会干扰 SAS，返回 401）。
- 后台轮询脚本会随主 agent turn 取消被杀 ⇒ **后台任务"失败" ≠ CI 失败**，续接时第一步直接查 `actions/runs` 结论，别重跑构建、别重推 tag。
- 竞态：CI 误触发旧代码后新/旧 run 竞争同 tag Release 资产（最后完成者赢）。触发前确认 remote head 已是目标 commit。
- PAT 存放：`C:\Users\Administrator\.workbuddy\mcp.json`（githubcopilot MCP 的 Authorization）。建议轮换细粒度 token，勿长期存 `.git/config`。

---

## 六、其他 Git 怪癖

| 现象 | 处置 |
|---|---|
| `git update-ref refs/remotes/origin/master` 静默失效（exit 0 不落盘） | `mkdir -p .git/refs/remotes/origin` 后直接写松散引用文件 |
| 远端出现 3 条 message 相同的同名提交 | MSG 从独立文件读（脚本里写死 `MSG` 会被派生脚本 sed 漏改）；推完 `GET /commits?per_page=3` 核对 |
| tree API 422 | 先查 remote ref 再重推 |
| tree 漏推新文件 | base_tree 只覆盖远端已有文件，新增文件必须显式加进 EXTRA 清单 |
| `git stash` 之外的对照实验 | 用 `cp -r` 到仓库外目录 |

---

## 七、失败模式

1. **信 exit code**：推送/部署 exit 0 ≠ 成功（Pages 25MiB 超限也 exit 0）→ 推后必须回读校验。
2. **手拼 sha / 用本地 sha 建远端 ref**：422。
3. **混用推送通道**：平行历史，仓库首页显示旧代码。
4. **手写文件清单**：漏 `pubspec.lock`、漏删除项 → tree 校验失败。
5. **在 heredoc 里写 `\n`**：YAML 解析失败、0 job 的 failure run。
6. **用 `git stash` 做实验**：毁库。
7. **只 commit 部分目录**：`reset --hard` 静默丢文件。
8. **不校验远端 tree 就推**：覆盖别人的成果。
9. **本地/远端字节数比对**：CRLF 假报警。
10. **CI 失败就改版本号重发**：历史通胀（正确做法见 02）。
