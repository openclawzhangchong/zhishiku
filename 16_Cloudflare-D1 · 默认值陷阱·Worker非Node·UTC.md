# 16 · Cloudflare + D1 后端：默认值陷阱 · Worker 非 Node · UTC

> **摘要**：一个跑在 Cloudflare Pages Functions + D1 上的真实业务后端，踩过的坑集中在四件事：**配置表读写的默认值陷阱**、**迁移文件的记账与行尾**、**绑定的开启方式与本地开发的费用陷阱**、**Worker 运行环境的差异（没有 Buffer、跑在 UTC）**。本文同时给出「一次性测试站」这套验证方法——用一个独立临时项目把不确定的技术点先跑通，再动主工程，这是本项目节省时间最多的一条经验。

**关键词**：Cloudflare Pages、Pages Functions、D1、wrangler、Workers AI 绑定、SQL 迁移、app_config、UTC、nodejs_compat、测试站、本地开发费用

---

## 一、配置表（app_config）：后台开关的唯一落点

### 结构

全项目的后台开关、凭据、模型选择**全部存在一张 `app_config` 表**里，不新建表、不新增 KV。读写就两个函数：

```js
await getConfig(env.DB, 'VOICE_INPUT_ENABLED');                       // 读，返回值可能是 null
await setConfig(env.DB, 'VOICE_INPUT_ENABLED', '1', Date.now());      // 写
```

**✅ 成功做法**：**不为一个新功能新建一张表**。配置项本质就是 K-V，用一张表 + `getConfig/setConfig` 包住，后台回显、前端下发、服务端校验三处都从同一处读。

### ⚠️ 头号坑：默认值的两种写法，语义完全相反

| 写法 | 库中无此 key 时 | 实际默认 |
|---|---|---|
| `(await getConfig(...)) === '1'` | 读到 `null` → `false` | **关** |
| `(await getConfig(...)) !== '0'` | 读到 `null` → `true` | **开** |

**同一个库里两种写法并存**（`DEPLOY_ENABLED` 用前者，`DEPLOY_QR` 用后者）。加新开关时如果照抄错了邻居，功能会**默认处于你没预期的状态**，而且老库升级后没有任何报错。

**✅ 防法**：新增配置项时在代码注释里写明「默认开 / 默认关」，并在 HANDOFF 里单独列一行。测试里钉死默认值断言（本项目 v1.38.0 就是因为**忘了写「默认关」的三条断言**，变异测试三条全绿放过，见 `22_流程质量`）。

### ⚠️ 二号坑：三个键要一起存，没传的保持原值

```js
export async function handleSetVoiceInput(request, env) {
  const body = await request.json().catch(() => null);
  if (!body) return json({ error: '请求体不是合法 JSON' }, 400);
  if (typeof body.enabled === 'boolean') { await setConfig(...); }
  if (body.mode === 'send' || body.mode === 'fill') { await setConfig(...); }   // 白名单校验
  if (body.model === 'turbo' || body.model === 'base') { await setConfig(...); }
  return json({ ok: true, /* 三行回显实际生效值 */ });
}
```

两个要点：
1. **每个字段独立判断，没传的不动**（否则一次只改一项会把另外两项清空）
2. **入库前做白名单校验**，库里不存完整模型名（`'turbo'` 而不是 `'@cf/openai/whisper-large-v3-turbo'`）——模型名变了不用改库

---

## 二、SQL 迁移：记账比写 SQL 更难

### 迁移文件的命名与内容

```
migrations/
  0001_*.sql
  ...
  0008_seed_public_mcp.sql     ← 播种公开 MCP 的工具别名表
```

`0008` 里那张别名表的形态（这是「外部工具名 → 中文名」的映射，给模型看）：

```sql
'{"get-tickets":"查余票","get-interline-tickets":"查中转余票",
  "get-train-route-stations":"查经停站","get-current-date":"查当前日期", ...}'
```

**⚠️ 坑 1：迁移是一次性的，跑过就不能改**

已经部署过的迁移文件再改，线上库不会重跑。发现写错了只能**再加一个迁移**。所以每个迁移提交前必须想清楚。

**✅ 记账做法**：`CHANGELOG.md` 里为每个迁移记一行「本版新增 migration 00XX，做了什么」，并在 HANDOFF 的「改了哪些文件」里列出。

**⚠️ 坑 2：Windows 上的行尾（CRLF）**

Windows 编辑器生成的 `.sql` / 脚本默认是 CRLF。跨平台工具对行尾敏感，表现为「文件内容看着完全一样，但校验/比对不过」。

**✅ 防法**：仓库根目录放 `.gitattributes`，对 `*.sql`、`*.sh`、`*.mjs` 显式指定 `text eol=lf`。本项目 `doubao/.gitattributes` 就是这个作用。

**⚠️ 坑 3：迁移里播种的数据会成为「隐性依赖」**

`0008` 播种的 12306 MCP 里**顺带**有一个 `get-current-date` 工具。**系统一度以为自己有日期能力**——实际上那是车票查询服务的附属工具，管理员一旦停用那个 MCP，整个系统就没有日期能力了（详见 `17_提示词`）。

> **凡是「能力」来自别人的服务，都要问一句：它关了会怎样。**

---

## 三、绑定（Bindings）：AI / D1 怎么开

### wrangler.jsonc 的写法

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "dog2027",
  "pages_build_output_dir": "./dist",
  "compatibility_date": "2026-09-29",
  "ai": { "binding": "AI" },          // Workers AI，语音识别用
  "d1_databases": [ /* ... */ ]
}
```

**✅ 已核实**：**Pages Functions 支持 AI 绑定，写 `wrangler.jsonc` 即可，不必去 Dashboard 点**。

### ⚠️ 本地开发的费用陷阱

```bash
npx wrangler pages dev --ai AI
```

**本地开发调 `--ai` 会真的调云端、产生真实费用。** 它不是模拟器。额度是整个 Cloudflare 账号共用的。

**✅ 防法**：需要验证 AI 相关逻辑时，**优先建一个独立测试站**（见第五节），而不是在本地反复跑主工程。

---

## 四、Worker 运行环境和 Node 不一样（两个必踩）

### 1. 没有 `Buffer`（除非开 `nodejs_compat`）

转 base64 要用 `btoa` + 分块：

```js
// Worker 里没有 Buffer（除非开 nodejs_compat），用 btoa 分块转
function bytesToBase64(bytes) {
  let s = '';
  const CHUNK = 0x8000;                       // ★ 必须分块，一次 apply 太大栈会爆
  for (let i = 0; i < bytes.length; i += CHUNK) {
    s += String.fromCharCode.apply(null, bytes.subarray(i, i + CHUNK));
  }
  return btoa(s);
}
```

**`CHUNK = 0x8000` 不是可选的**——`String.fromCharCode.apply` 参数个数有上限，几十 KB 的音频一次性传会抛异常。

### 2. 跑在 UTC 上（不是北京时间）

```js
const d = new Date(Date.now() + 8 * 3600 * 1000);   // ★ 手动 +8，正确
```

**直接 `new Date().getDate()` 会在北京时间 00:00–08:00 给出昨天**，一天里错 8 小时，而且极难发现。

**验证方法**（这条要背下来）：

```bash
node -e "
const t = new Date('2026-10-05T17:30:00Z');   // UTC 17:30 = 北京 10-06 01:30
console.log('直接 toISOString:', t.toISOString().slice(0,10), '<- 会得到 10-05 ❌');
const bj = new Date(t.getTime()+8*3600*1000);
console.log('手动 +8 小时:    ', bj.toISOString().slice(0,10), '<- 应得到 10-06 ✅');
"
```

---

## 五、★ 一次性测试站：本项目省时间最多的一条经验

### 做法

遇到「不确定的技术点」（比如「微信内置浏览器能不能录音」「whisper 认不认 webm/opus」），**不猜、不在主工程里试**，而是：

1. 建一个**独立的最小项目**（例：`voice-test`，部署到 `voice-test-*.pages.dev`）
2. 只做这一件事，做到能跑通
3. 拿到实测数字
4. **用完随时可删**，跟主工程零耦合

### 实测产出（语音输入那次）

```
POST /api/stt  content-type: audio/mpeg  30816 字节
→ ok:true  step:done  audioKind:base64  detectedLang:zh
  text: "你好,这是一段语音识别的测试,请把这句话识别出来。"   ← 一字不差
  aiMs: 2424ms

小米浏览器 / Android 16：
MediaRecorder: true   getUserMedia: true
audio/webm;codecs=opus = true   audio/webm = true   audio/mp4 = true
audio/ogg / audio/wav / audio/mpeg = false      ← 只支持 webm 和 mp4
录音 5.50s → 75094 字节 → HTTP 200  往返 1352ms（其中 AI 937ms）
text: "在吗在吗这是一个测试的语音测试能不能语音输入"   ← 一字不差
```

### 为什么这条值钱

| 不做测试站 | 做测试站 |
|---|---|
| 在主工程里改 250 行才发现问题 | 40 行验证完，实施时**不用再验证格式** |
| 失败要回滚、要清理 | 删掉项目即可，零残留 |
| 猜测 → 争论 → 返工 | 数字 → 决策 |

**✅ 判定规则**：**凡是「我不知道它行不行」的技术点，一律先建测试站。** 成本 10 分钟，省下的通常是半天。

---

## 六、路由分发：一个 `src/index.js` 管全部

```js
// 用户侧
if (m === 'POST' && resource === 'stt' && !id) return handleStt(request, env, userId);
// 后台侧
if (m === 'POST' && aid === 'voice-input') return handleSetVoiceInput(request, env);
```

**约定**：`resource` 是用户资源名，`aid` 是后台动作 id，`id` 是资源主键。三个维度组合出全部路由。

**⚠️ 坑**：新增路由时**必须同时改三处**——路由分发、后台回显（`handleGetConfig`）、前端下发（`GET /api/config`）。漏了回显会表现为「后台保存成功但刷新后又变回旧值」。

---

## 七、失败清单

| # | 失败模式 | 表现 | 防法 |
|---|---|---|---|
| 1 | 默认值写法照抄错邻居 | 功能默认状态与预期相反 | `=== '1'` / `!== '0'` 对照表 + 断言 |
| 2 | 一次性保存接口清空未传字段 | 改一项，另外两项归零 | 每字段独立判断 |
| 3 | 迁移跑过后再改 | 线上库不生效 | 只加不改 + 记账 |
| 4 | CRLF 跨平台 | 内容看着一样但校验不过 | `.gitattributes` 指定 eol=lf |
| 5 | 「能力」来自外部服务 | 服务一停系统就缺能力 | 关键能力必须内置 |
| 6 | 本地 `--ai` 反复调 | 真花钱 | 用测试站 |
| 7 | Worker 里用 Buffer | 报错 | `btoa` 分块（CHUNK 0x8000） |
| 8 | Worker 里直接取本地日期 | 每天凌晨 8 小时错一天 | 手动 +8 小时 |
| 9 | 加路由只加一处 | 后台保存后刷新回旧值 | 分发 / 回显 / 下发三处同改 |
| 10 | 在主工程里试不确定的技术 | 返工 + 回滚 | 一次性测试站 |

---

## 八、一句话总结

> **Cloudflare 这套东西本身不难，难的是「配置默认值的语义」和「Worker 不是 Node」这两件事。**
> 前者会让功能静默处于错误状态，后者会让代码在本地跑得好好的、上线就错。
> 而**一次性测试站**是应对所有「我不知道行不行」的通用解——它把争论变成数字。
