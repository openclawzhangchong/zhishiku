# 10 · MCP 与云端部署：tools/call 实测 · Pages 25MiB · KV 绑定

> **摘要**：MCP 接入有"协议通 / 清单通 / 业务通"三级，只验到第二级就会漏掉余额为 0、key 错误这类问题。本篇记录 MCP 配置与排查、自研 Worker 骨架、Cloudflare Pages/Workers/KV/D1 的绑定与部署坑，以及阿里云 Windows 服务器的端口与自启纪律。
>
> **关键词**：mcp.json、信任、tools/list vs tools/call、Accept 双值、UA 校验、大产物短码、env 守卫、Pages 25MiB、KV 绑定名、routes 段序、阿里云端口
>
> **来源**：mcpnew、2026-09-09、2026-09-14（阿里云知识库）、2026-09-29（工作台）、xinxibook（worker-mcp）

---

## 一、MCP 配置与验证

### 1.1 配置文件位置（最容易踩的第一坑）
- 用户级自定义 MCP 写在 **`C:\Users\Administrator\.workbuddy\mcp.json`** 的 `mcpServers` 节点。
- **不是** `.mcp.json`（带点）；**不要改** `~/.workbuddy/connectors/default/mcp.json`（市场托管，会被覆盖）。
- 改完必须到连接器管理页「自定义连接器」点**信任**，必要时重启客户端。

### 1.2 三级验证（逐级才算通过）
| 级别 | 动作 | 判据 |
|---|---|---|
| **L1 协议** | `initialize` | 返回 `protocolVersion`、`capabilities.tools`、`Mcp-Session-Id` |
| **L2 清单** | `tools/list` | 工具数量与命名符合预期 |
| **L3 业务** | 真实 `tools/call` 一次 | 返回正常结果；**中文参数要实测**（如用"泉州市华侨职业中专学校"取经纬度） |

> **`tools/list` 通过只是协议层通过**，必须再打一次 `tools/call` 才算接入完成。429/402 属业务态（余额不足），不是配置错。

### 1.3 各家失败形态不同（逐个记）
| 服务 | 失败形态 |
|---|---|
| amap | **HTTP 200 + body `{"status":"0","info":"INVALID_USER_KEY"}`**；`Accept` 缺 `text/event-stream` → 406 |
| biji-notes | 鉴权失败 401 + `{ok:false,error:{...}}`；**CF + 源站双层 UA 校验**（Flutter/curl/okhttp UA → 401，空 UA → CF 403 Error 1010）→ 必须设浏览器式 UA |
| zhipu 搜索 | 握手/鉴权/`tools/list` 全 OK（3 个工具），`tools/call` 返 **429「余额不足或无可用资源包」** |
| 自建 Worker | 未先 `initialize` → `No valid session id`；`ping` 返回 `method not found` → 每 30~60s 断（必须回 `result:{}`） |

- **绝不能只用 HTTP 状态码判成功**。
- **凭据两侧禁止留白**（URL query 里 `?Authorization= <key>` 多一个空格 → 鉴权失败）；能放 header 就别放 URL（URL 会进日志）。
- 需要长耗时的加 `"timeout": 60000`。
- Streamable HTTP 型：凭据放 `headers.Authorization`；SSE 型：常放 URL query。

---

## 二、自研 MCP 服务端最小骨架（Worker 版已验证）

```
jsonResponse() 统一带 CORS
  → OPTIONS 短路 204
  → Bearer 校验（env.MCP_TOKEN 优先，默认口令兜底）
  → JSON-RPC 2.0 分发：initialize / tools/list / tools/call / ping(返回 result:{})
  → 大产物存 KV 回短链
```

| 要点 | 说明 |
|---|---|
| CORS | 所有响应（**含错误响应**）统一：`Allow-Origin: *`、`Allow-Methods: POST, GET, OPTIONS`、`Allow-Headers: Content-Type, Authorization, Mcp-Session-Id, MCP-Protocol-Version`。**必须显式处理 OPTIONS**，写进 `jsonResponse()` 统一出口 |
| env 守卫 | `if (!env || !env.DL_KV) return null;` 调用方**降级 base64**，`mode: url/base64/both` 三态。**所有 `env.XXX` 访问前必须守卫**，绑定缺失走功能降级，绝不 throw |
| token | `var token = (env && env.MCP_TOKEN) || <默认>`，生产用 `npx wrangler secret put MCP_TOKEN` 覆盖 |
| 大产物 | 31 万字符 base64 会撑爆 AI 客户端 → **存 KV（24h TTL）+ 返回 6 位码短链** `GET /api/mcp/dl/<码>`（免鉴权，码即钥匙）→ 响应降到 632 字符 |
| 复用前端代码 | `globalThis.window = globalThis` shim 把浏览器端 `engine.js` 直接搬进 Worker，`build_mcp.js` 拼单文件 |

---

## 三、Cloudflare 部署

### 3.1 Pages
- **单文件上限 25 MiB**：`dist/` 里躺着 133MB EXE → 部署"看似成功（exit 0）"但线上没更新，报错被 tail 截断。
  - 修法：`mkdir _pages_deploy && cp dist/index.html _pages_deploy/ && npx wrangler pages deploy _pages_deploy`（`_pages_deploy/` 进 `.gitignore`）。
  - **不能用 exit code 判部署成功**，部署后必须回读线上内容校验。
- 建项目 POST 必须带 `production_branch`；PATCH deployment_config 设 D1 绑定**不生效** → 用 **Pages 模式 wrangler.toml**（`pages_build_output_dir` + `[[d1_databases]]`）+ `wrangler pages deploy`，**绑定随 deployment 走**（settings 显示 null 但运行时 `env.DB` 有值）。
- **项目名 ≠ 域名**：项目名 `xiaole-web`，域名 `xiaole-web-1nx.pages.dev`（CF 自动加后缀）；`wrangler deploy` 用域名后缀会报项目不存在。
- Pages Functions：**路径 = 文件目录**（`/api/data` → `functions/api/data.js`，不能写 `functions/api.js`）。
- KV **绑定名必须与代码 `env.XXX` 完全一致**（`DIANMING_KV` vs `ROLLCALL_KV` 不一致 → 读不到）。

### 3.2 Workers
- `workers.dev` 域名本机不可达（超时 + 502）→ 需要长期域名就绑自定义域：`routes=[{pattern="nav-api.jiuyeke.cn", custom_domain=true}]`。**`routes` 必须写在 `[vars]` 之前**，否则被解析成 vars 的一项。
- 新路由部署后 `curl` exit 35（SSL 握手失败）→ **证书传播约 10s 自愈**，别误判为部署失败。带 `/*` 的通配要单独写一条路由。
- 域名务必核准（`jiuyeke.cn` 不是 `jiuyuke.cn`，历史记忆拼错过）。
- CF 防火墙拦 `Python-urllib` 默认 UA（403）→ **脚本发请求一律自定义 UA**。

### 3.3 选型理由（网页版 CF Pages + Workers + D1）
| 组件 | 职责 | 理由 |
|---|---|---|
| CF Pages | 静态托管单文件 `app.html` | 零成本、全球 CDN、直接拿 `*.pages.dev` |
| Workers | `/ws` 转头代理、`/ota`、`/mqtt` TCP 桥、`/mcp-proxy`、`/dev-check`、`/logs`、`/memory-sync` | **浏览器不能自定义 WS 握手头**（Authorization/Device-Id）→ 必须代理补头；**浏览器不能连裸 TCP 8883** → MQTT 必须走桥 |
| D1 `xiaole-cloud` | users / chat_blocks | 与 Workers 同栈免运维、边缘就近读 |
| KV `LOGS_KV` | 运行日志 | 写多读少、可接受最终一致，比 D1 便宜 |

**云同步"三容器"设计**（互不干扰的结构性保证）
| 容器 | 语义 | 写入方式 |
|---|---|---|
| `settings_json` | 整机快照 | **整块覆盖** |
| `defaults_json` | 租户默认包 | 整块 |
| `tokens_json` | 账号级令牌 | **逐键合并**，存原值 |

---

## 四、阿里云 Windows 服务器运维

| 事项 | 做法 |
|---|---|
| SSH 三连坑 | `setsid` 不可用；`/tmp` 跨调用丢失；`powershell` 被安全策略拦截；多行脚本"无法识别命令根" → 用 `SSH_ASKPASS=askpass.sh` + `SSH_ASKPASS_REQUIRE=force` + `DISPLAY=:0` 免密；临时文件放工作目录；多行合并成单行 |
| 端口盘点 | 部署前 `netstat -ano \| findstr LISTENING` + `tasklist`（既有占用 80/443/8081/8443、7000/8080、4000）→ 新服务选空闲端口 |
| 防火墙 | `netsh advfirewall firewall add rule`；**安全组 + Windows 防火墙双放行** |
| 自启 | `schtasks /Create`（SYSTEM、RunLevel Highest） |
| 重启边界 | 改 `server.js` 必须 `schtasks /End /TN ApkHost` 再 `/Run`；**改 `public/index.html` 不用重启**（每次请求读盘）——这个区别要写进交接文档 |
| 依赖最小化 | apkhost 只用 Node 内置模块；静态资源全下到 `static/vendor/`（内网离线可用） |

---

## 五、内部 Web 工作台做法（班主任材料审核工作台）

**数据模型**
```
cohort(届, is_active 唯一为1) → class_group → student(学号 UNIQUE per cohort, system_status)
                                            ├─ submission        (A 文件)
                                            ├─ internship_record (B 结构化)
                                            └─ manual_status     (C 状态机)
material_type(code 四类: agreement/tracking/internship/manual, subtype A/B/C)
review_log(stage=ai_review|human_recheck) / ai_review(confidence) / audit_log
```
- **多届切换**：所有业务表带 `cohort_id`；看板/报表/审核默认作用于当前届，历史届只读。
- 完成率分母口径**写进文档**（剔除 `dropped`/`graduated`，`drop_pending` 仍计入）。
- **红线**：终态 `locked=1`，后端统一拒绝改删；纠错走超管解锁或复议申请。
- 名册导入：**以学号为唯一键增量更新**，56 列中 6 个核心字段建列，其余存 `raw_data` JSON。

**导出与权限**
- 月度 Excel（openpyxl）固定**双 sheet**：`班级汇总` + `学生明细`；手动导出 + 每月自动存档双轨，每次写 `report_export` 可查历史。
- 角色只有 admin / teacher；**自助注册 + 认领班级 → 管理员审核 → 生效**；靠角色 + `class_scope` 装饰器隔离。
- 文件下载/预览接口 `/api/files/{id}` **必须带鉴权防越权**（不带 token → 401，APK 托管踩过）。
- 公网入口必须补：强 admin 密码、Access、每日备份。

**前端**
- Flask + Jinja2 + SQLite + 本地目录；Bootstrap/pdf.js/jsPDF/Chart.js **全部下载到 `static/vendor/`**，不依赖 CDN。
- **API 优先**：网页与 `/api/*` 共用同一 `services/` 层 → MCP/Agent 能直接复用业务逻辑。
- 上传压缩**优先浏览器端**（canvas / pdf.js→canvas→jsPDF），Ghostscript 仅服务端兜底（保文字层）；不留存原文件 ⇒ 压缩前必须给预览 + 确认。
- 审核台：左预览 + 右批注 + 快捷键 `←/→` 翻份、`Y` 通过、`N` 退回；**不做批量通过**（质量优先）。
- 实施顺序：骨架登录 → 名册班级 → 文件闭环（最痛的先拿到手）→ 备案表统计 → 看板报表 → MCP + AI 初审 → 自测部署。

---

## 六、失败模式

1. **"测试通过但任务失败"**：只测连通性不测业务（`httpx.AsyncClient` 把 Authorization 固化，Web UI 改 token 不生效，测试按钮 new 客户端所以显示正常）→ 头每次动态生成 + **token 指纹**（`len=N sha256:前8位`）核对。
2. **拿中间态当结果**：端口 listen 但没 `serve_forever`；页面显示"已启用 20/20"实际下发 0。
3. **混用 git 推送通道** → 平行历史（见 03）。
4. **凭 exit code 判部署成功** → Pages 25MiB 静默失败。
5. **把凭据明文写进文档还同步云端** → 至少加安全警告 + 轮换建议。
6. **只复制部分代码做验证** → 漏掉加载期崩溃，误诊为"浏览器缓存"。
7. **只看 HTTP 状态码** → amap 200 藏 INVALID_USER_KEY、智谱 list 200 但 call 429。
8. **前端无超时 / 无条件绑定** → 永久 pending 与整页崩溃。
9. **中途换栈**（专家建议 Java/Vue 就换）→ 与既有资产冲突。
10. **部署不盘点端口** → 顶掉既有服务。
