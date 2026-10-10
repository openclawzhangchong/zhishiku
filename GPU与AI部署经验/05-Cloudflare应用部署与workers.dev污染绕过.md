# 05 · Cloudflare 应用部署与 workers.dev 污染绕过

> **摘要**：记录 Cloudflare Workers/Pages 应用（Second Brain v3.1.0）的部署全过程：代码审计结论、wrangler 部署阻塞点、`*.workers.dev` 被福建 ISP 选择性 DNS 污染后用 Pages 反代绕过的发现、中文 embedding 切换、Vectorize metadata filter 静默失效、网关限流与 Web 控制台中文化。目标是不在本机网络特殊环境下重复踩部署坑。
>
> **关键词**：Cloudflare；Workers；Pages；wrangler；workers.dev 污染；Pages 反代；Vectorize；bge-m3；i18n 中文化；D1；KV；AUTH_TOKEN

---

## 一、踩过的坑（吃过哪些亏）

1. **`*.workers.dev` 被选择性 DNS 污染**：DNS 解析到 Facebook 段（157.240.x），HTTPS 000/21s 超时。一度下“六条路全断”的错判，其实没测兄弟域名 `*.pages.dev`。
2. **`wrangler secret put AUTH_TOKEN` 报 `10034 You need to verify your email`**：用户去 QQ 邮箱点验证邮件才能继续；`/client/v4/user` 不返回 verified 字段，只能靠重试判断。
3. **`/recall` 是 GET 不是 POST**：之前用 POST 打 `/recall` 一直 "Not found"；参数 `?query=`，`/capture` 才是 POST。
4. **Vectorize metadata filter 静默失效**：带 `workspace_id` filter 查到 0 条，`$eq` 直接报 40030；filter 返回空集合而非抛异常 → 语义臂从未生效、去重/矛盾检测一直失效。
5. **i18n 生成脚本 `+` 优先级坑**：`pad + '  ' + regex.test(k) ? k : q(k)` 中 `+` 高于 `?:` → key 被拼成 `authnavhomerecall...`；`new vm.Script()` 语法检查查不出这种语义错误 → 必须实际执行取值验证。
6. **中文化只加中文会挂测试**：`test/ui/i18n.test.ts` 要求每个 key 在 en/it/zh 三 catalog 都齐；手改 `i18n.js` 里的 `I18N_ZH` 会被 `gen-zh-catalog.js` 覆盖。
7. **`wrangler pages deploy` 不支持 `--kv`**：绑定只能写进 `wrangler.toml` 的 `[[kv_namespaces]]`。
8. **`wrangler kv key list/get` 默认读写本地副本**：查远程必须加 `--remote`，否则误判“限流没生效”。
9. **默认英文 embedding 中文召回为空**：`bge-small-en-v1.5` 是英文模型，中文提问命中不了 → 必须切 `bge-m3`（1024 维）。

## 二、怎样才能成功（成功做法）

- **污染绕行 = Pages 反代**：`*.pages.dev` 未被污染（DNS 104.18.21.135，HTTPS 301/0.96s ✅）。`npx wrangler pages project create second-brain-gateway --production-branch=main`（Pages 项目不能靠 `pages deploy` 自动创建，必须先 create）→ `functions/[[path]].ts` 全量反代到源站 Worker（剥离逐跳首部、改写 Host、`body` 转发需 `duplex:'half'`、去 `content-encoding`）→ `npx wrangler pages deploy public --project-name=second-brain-gateway --branch=main`。客户端只换域名。
- **下结论前先测对照组兄弟域名**：之前只测 workers.dev 就下“全断”，其实 pages.dev 通。
- **中文 embedding 切 bge-m3**：`POST /accounts/{id}/ai/run/@cf/baai/bge-m3` 直接验证维度（1024）；新建 `second-brain-vectors-1024` → `PATCH /config {"EMBEDDING_MODEL":"@cf/baai/bge-m3"}`（运行时 config，热切换无需重编译）→ 改 `wrangler.jsonc` 绑定 → deploy；中文必须全量 re-embed。
- **Vectorize filter 静默失效修法**：加 `VECTORIZE_FILTER_ENABLED=false` env，在 `search.ts`+`duplicate.ts` 禁用 filter；别一上来改 API 语义，用 env 开关避免破坏上游测试。
- **中文化铁律**：补 key 必须 en/it/zh 三语言同步；`zh` catalog 由脚本生成、勿手改；en/it 直接在 `i18n.js` 改后 `grep -n` 复查落点。中文无复数 → `.one`/`.other` 填同一字符串。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **wrangler 4.x 不支持账号密码登录**，只支持 OAuth（`npx wrangler login` 浏览器）。
- **D1/KV 免费计划区域固定 WNAM 不能改**；已用满免费计划 5 个 cron。
- **Pages 不是 zone 挂不上 WAF** → 限流只能做应用层（KV `RATE_LIMIT_KV`：240 次/分/IP；认证失败 8 次/分 → 封 15 分钟）。
- **本机无 `gh` CLI**；`npm install` 该仓库约 5.5 分钟（271 包）；`wrangler dev --remote` 本机崩溃、本地 Workers AI embed 报 `InferenceUpstreamError`、代理不通 → 本地跑不了完整链路；`wrangler tail` 走 WebSocket 本机代理 502 看不了实时日志。
- **安全清理**：`DEPLOY.md`/`scripts` 硬编码真实 AUTH_TOKEN → 全改占位符/环境变量；`.gitignore` 排除 `node_modules/.dev.vars/.env/.wrangler/*.pem`；明文 classic PAT `ghp_...`、Cloudflare 密码务必吊销/改密 + 开 2FA。

## 四、具体怎么实现（实现方式）

**1. 代码审计结论（v3.1.0）**：`npx wrangler types && npx tsc --noEmit` 0 错误；`npx vitest run` 3572 通过 / 3 失败 / 7 skipped（3 失败全是 Windows 路径分隔符，非产品缺陷）。主要缺点：静态 AUTH_TOKEN + 无盐 SHA-256 + 无限流 + CORS `*`；默认英文 embedding；Cloudflare 深度锁定且用满免费 cron；自动化分类/去重/压缩不可控；D1 明文 + 删除链路非原子。

**2. 端到端冒烟（绕过污染后全绿）**：`/health` 200；`/capture`×2 写入成功；`/recall?query=数据库为什么选这个` 命中 score 100；`/mcp` SSE 握手成功；`/stats` count=2 unvectorized=0。

**3. Web 控制台中文化（556 条，三语言 100% 对齐）**：新增 `I18N_ZH`（556 条），默认语言改 zh；`index.html` 124 处硬编码英文用 catalog 脚本批量替换；`main.css` 字体栈追加中文字体；`localize-html.js` 按 `data-i18n` + `data-i18n-attr` 把属性值换中文（首轮漏 33 处）。

**4. 打包成私有仓库**：产出 README(45KB) + DEPLOY.md(30KB) + AGENT-INTEGRATION.md(31KB) + VERSION.md + docs/ + source/(525 文件) + gateway/ + patches/ + scripts/。推送走 Git Data API（本机 `github.com:443` 被 reset）。
