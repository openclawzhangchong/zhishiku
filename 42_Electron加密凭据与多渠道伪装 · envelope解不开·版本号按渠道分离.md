# 42 · Electron 客户端的加密凭据与多渠道伪装：venvelope 解不开 · 版本号按渠道分离

> **摘要**：新版 Electron 客户端（WorkBuddy/CodeBuddy 5.6+）把 token 用**本机运行时密钥**加密落盘（`$wbEncrypted` 信封），密钥不落盘、外部进程复现不了——从客户端"导入"的账号只能切换、不能出站。要用它做 API 反代，唯一出路是**自己走 OAuth 拿明文**。另外同一个产品的国内版/国际版是两个不同发行物，版本号长期不同步（实测差十几个版本），伪装参数必须按渠道分离。

**关键词**：wbEncrypted, envelope, 加密凭据, safeStorage, DPAPI, AES-256-GCM, HKDF, keyblob, static-v1, at-rest encryption, 反代 503, 11128, 11102, 11158, first message is not system prompt, 伪装, User-Agent, 版本分离, Electron, app.asar

---

## 主题 1：加密信封凭据解不开——这是设计，不是 bug

| 项 | 内容 |
|---|---|
| **症状** | 账号在库里、UI 显示已登录，但反代一出站就失败；界面红字「该账号凭据为加密存储，仅支持切换账号；查询积分需重新登录」。账号 JSON 里 `access_token` 是 `{"$wbEncrypted":1,"envelope":"eyJzdWl0ZSI6..."}` |
| **错误归因** | 先以为是 UA 版本号不对（换了几轮版本号、补了一堆头，仍不通） |
| **证据链** | ① 账号库 `access_token` 是对象不是字符串 → 密文；② 同一库里另一个号是明文 JWT → 那个号能出站；③ 解 envelope：base64 JSON `{suite:1, keyId, nonce, authTag, ciphertext}` = AES-256-GCM；④ asar 里找到 `CredentialFieldCipher`：`aesKey = HKDF-SHA256(masterKey ‖ userId, salt)`，AAD = `"WB-AAD\0" ‖ userId`；⑤ `masterKey` 来自 `~/.workbuddy-ai/keyblob` 的 `static-v1` slot，由 `AtRestCrypto` 解开，其对称密钥由**运行时 bootstrap 注入**、磁盘上无明文 |
| **根因** | 密钥链最终锚在一个不落盘的运行时密钥上。反编译能看懂算法，但**复现不了密钥**——这是刻意的安全设计 |
| **为什么没早发现** | 之前用旧缓存里的明文 token 测通过，误以为链路没问题；实际库里那个号早已被客户端加密存储 |
| **修复** | 不去破解。改为：**加密凭据账号排除出反代池**（否则"有账号但每次出站都失败"把入口拖成 503），并在 503 文案里明确告诉用户「请重新登录拿可出站凭据」 |
| **可复用教训** | 看到 `{"$wbEncrypted":...}` 这类信封就别想着解密——直接走产品自己的 OAuth 流程拿明文；并把这类账号从"要出站"的池子里剔除，不要让它污染可用性判断 |

**验证状态**：真机实测（加密账号出站必失败；明文账号出站 200）

### 拿明文的正解：走自己的 OAuth

```bash
# 1) 申请 state（platform 参数决定拿哪一档的登录态）
POST {api}/v2/plugin/auth/state?platform=workbuddy-ai
   → { state, authUrl: "{api}/login?platform=workbuddy-ai&state=<uuid>" }

# 2) 用户在浏览器打开 authUrl 完成授权

# 3) 轮询换明文 token（11217 = 还没授权）
GET {api}/v2/plugin/auth/token?state=<uuid>
   → { data: { accessToken, refreshToken, expiresIn } }
```

## 主题 2：同一产品的国内版/国际版是两套身份，伪装必须分离

| 项 | 内容 |
|---|---|
| **症状** | 国际版一直"能用"但实际在冒充国内版的版本号；两档共用一个 `DEFAULT_CLIENT_VERSION` 常量 |
| **证据链** | ① 两个 exe 都在本机：`WorkBuddy.exe` FileVersion **5.11.3**、`WorkBuddyAI.exe` **5.9.4**；② 各自内置 `@genie/agent-cli`：**2.176.0** vs **2.165.0**；③ 两个 asar 的 API 域名、路由表都不同 |
| **根因** | 把"同一产品"当成"同一种客户端"。实际是两条独立发版线，差十几个版本；上游只要做"这个版本号在你的渠道出现过吗"的时间线校验就会露馅 |
| **修复** | 版本常量按 edition 分离（`masquerade_client_version(edition)`），UA 与 `X-IDE-Version` 都取账号所属档位的值；加防回归单测断言两档**必须不同** |
| **可复用教训** | 伪装参数（版本号、UA、头集合、可用模型）一律**按渠道/发行物分离**，不要共用全局常量；取值从各自安装目录解包，不要跨渠道抄 |

**验证状态**：真机实测（两档各自 200；上游目前不校验精确版本号，但分离后更稳）

### 国际版独有的额外坑（同一轮踩到的）

| 现象 | 原因与解法 |
|---|---|
| `/v2/enterprises/personal/models` → **404 Route Not Found** | 国际版网关**没有**模型列表路由；模型清单只存在于客户端本地 `~/.workbuddy-ai/cache/acc-product-config-v3-*.json`。反代拉不到时**降级**返回已知可用模型，别整条入口 502（客户端一上来先拉 `/v1/models`，拿到 502 就放弃） |
| `deep-model` → **11102** model service info not found | 配置里列了但服务端不认；只把**实测通过**的模型名写进清单 |
| `primary-model` → **11158** | 该模型不允许免费用户 |
| **11128** `first message is not system prompt` | 国际版强制首条消息是 `system`；客户端不带时自动兜一条默认 system |
| `acp-connection-id` / `traceparent` / `b3` / `X-B3-ParentSpanId` | 真机有、去掉仍 200（非强制），但补齐更贴近真机 |

> 关联：`39_伪装头对齐与发版兜底`（国内版伪装与发版兜底全套）。
