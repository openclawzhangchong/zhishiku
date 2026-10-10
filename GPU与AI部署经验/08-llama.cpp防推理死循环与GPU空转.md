# 08 · llama.cpp 推理死循环与 GPU 空转事故（给推理服务加刹车）

> **摘要**：记录 2026-10-09 本机 `llama-server` 一次真实的 **GPU 空转事故**：模型陷入推理死循环，客户端断开后服务端**不中止生成**，而 `--n-predict` 默认 `-1`（无上限），于是一次被遗弃的请求把 V100 满载烧了 **34 分钟 / 21.8 万 token / 160 W**。本篇给出三步定位法（`nvidia-smi` → `/slots` → `netstat`）、两个刹车参数（`-n` + `--reasoning-budget`）的配置与验证方法，以及该构建的几个反直觉陷阱。**核心结论：`llama-server` 默认是「无上限 + 不响应断连」的危险组合，必须主动加刹车。**
>
> **关键词**：llama.cpp；推理死循环；GPU 空转；n-predict；reasoning-budget；reasoning-preserve；/slots；netstat；事故复盘；Qwen3.6；MoE

---

## 一、踩过的坑（吃过哪些亏）

1. **客户端断开后服务端继续生成**：浏览器关掉、脚本被 Ctrl-C，`llama-server` 照样算完。实测断开后仍持续生成 **34 分钟**、产出 **21.8 万 token**，直到把 `n_ctx`（262144）塞满才停。
2. **`--n-predict` 默认是 `-1`（无上限）**：没有刹车。日志里 `/slots` 的 `n_remain: -1` 就是这个状态的标志。
3. **误判循环的触发条件**：一开始以为「问它运行在哪张显卡上」这个问题会触发循环，**错了**。对照实验证明：同样的问句，全新单轮请求 3.7 秒干净答完（仅 41 token 思考），而在带累积历史的 WebUI 里才会死循环。**循环由上下文状态诱发，不是问题本身。**
4. **`/slots` 的动作接口是空壳**：`POST /slots/0?action=erase` 和 `?action=cancel` 均返回 **501 Not Implemented**，杀不掉正在跑的生成，只能重启进程或等它撞上限。
5. **该构建不写请求日志**：`llama-server.log` 只记启动过程，**不记录请求**。所以「有没有人在调用」不能翻日志，必须看 `/slots` + `netstat`。
6. **`/props` 会骗人**：即使 `-n 32000` 已生效，`default_generation_settings.params.n_predict` 依然显示 **`-1`**。据它判断参数没生效会走弯路。

## 二、怎样才能成功（成功做法）

- **加两个刹车，一次到位**：
  - `-n 32000` —— 单次输出封顶，把最坏情况从「无限」压到「约 4 分钟」（~138 tok/s）
  - `--reasoning-budget 8000` —— 思考段落封顶，用尽即注入结束思考标签、强制转入正式回答，循环无法无限延续
- **三步定位法定性**（见第五节）：`nvidia-smi` 看谁在占卡 → `/slots` 看 `is_processing`/`n_decoded` → `netstat` 看有没有客户端。**第二步和第三步一对照就能定性。**
- **验证 `-n` 生效要看 `/slots` 的 `n_remain`**，不要看 `/props`：`n_remain` 应精确等于 `-n` 值减去 `n_decoded`。实测 `n_decoded + n_remain` 恒等于 32000。
- **验证思考预算机制**：该构建支持**请求级**覆盖 `thinking_budget_tokens`。用它验证同一套机制：设 50 时 reasoning 从均值 1989 字符降到 **171 字符**（3 次一致）→ 机制确实工作。
- **重启是唯一可靠的「立即停止」手段**：`taskkill /f /im llama-server.exe` → `schtasks /run /tn "llamacpp-server"`。加载约 50~70 秒，期间所有接口返回 **503 `Loading model`**（是「还在加载」，不是故障）。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **`-n` 只是「默认值」，不是硬上限**：实测客户端显式传的 `max_tokens` 会被**精确遵守**（发 64 停 64，发 300 停 300）。WebUI 不发这个字段所以被管住，**但客户端自己传更大的值就能绕过** —— 客户端里的「最大输出」也要设合理值。
- **循环的成因是 `--reasoning-preserve`**：服务端默认把推理内容保留进后续轮次，长推理链自我强化 → 循环。**根治首选 `--no-reasoning-preserve`**，比 `--reasoning-budget` 更直击成因。日志里有明确告警：
  ```
  chat template supports preserving reasoning, it is enabled by default
  (may use more tokens, disable via --no-reasoning-preserve)
  ```
- **`reasoning-budget` 与 `max_tokens` 共享额度**：思考 token 计入输出预算。`budget=2048 + max_tokens=900` 时，复杂题会把 900 全耗在思考上、正文 0 字。给推理模型设 `max_tokens` 不要低于 2048。
- **`-n` 不影响显存**：KV 缓存按 `-c` 预分配，与 `-n` 无关，调大 `-n` 不会多占显存，只会拉长最坏空转时长。
- **运行实例会与启动 bat 脱节**：手工启动的实例参数与 bat 完全不同（本次实测：手工实例既没带 `-n` 也没带 `--reasoning-budget`，模型文件也不是 bat 指定的那个）。**判断「当前跑的是什么」必须看 `/props` 的 `model_path` + 系统里的真实命令行，不要相信 bat 的内容。**

## 四、具体怎么实现（实现方式）

**1. 参数配置**（`start-llama-server.bat`，纯 ASCII + LF 行尾）

```bat
set "NPREDICT=32000"
set "THINKBUDGET=8000"
...
llama-server.exe -m %MODEL% ... --min-p 0 -n %NPREDICT% --reasoning-budget %THINKBUDGET% ...
```

用 `set "X=值"` + `%X%` 展开，避免在 `cmd /c "..."` 里出现嵌套引号（嵌套引号会截断外层字符串）。

**2. 验证 `-n` 生效**（发一个**不带 `max_tokens`** 的长请求，中途查槽位）

```bash
curl -s -H "Authorization: Bearer <APIKEY>" http://127.0.0.1:8080/slots
# 判据: n_remain 精确等于 32000 - n_decoded
```

实测：`(447,2053) (638,1862) (824,1676) (1012,1488) (1172,1328) (1424,1076)` —— 每组相加恒为 32000。

**3. 验证思考预算机制**（请求级覆盖，无需重启）

```json
{"model": "...", "messages": [...], "max_tokens": 32000,
 "thinking_budget_tokens": 50}
```

实测 reasoning 字符数：基线 1989 → `=50` 时 **171**（3 次一致）。`=0` 视为不限制；`=5000` 高于自然思考量故不截断。
> 附带收获：`thinking_budget_tokens` 是**每个请求都能单独指定**的，客户端可按需调节（简单问答 512、复杂任务 8000）。服务端 `--reasoning-budget` 只是「客户端没指定时的默认值」。

**4. 参数取值参考**

| 参数 | 取值 | 含义 | 最坏空转时长（~138 tok/s） |
|---|---|---|---|
| `-n` | 16000 | 更保守 | ~2 分钟 |
| `-n` | **32000** | 本次采用 | ~4 分钟 |
| `-n` | 65536 | 更宽松 | ~8 分钟 |
| `--reasoning-budget` | **8000** | 本次采用 | ~1 分钟 |

## 五、三步定位法（下次 GPU 忙但「没人用」时照做）

**第 1 步 · 看显存和进程**

```bash
nvidia-smi --query-gpu=utilization.gpu,power.draw,temperature.gpu,memory.used --format=csv
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
```

**第 2 步 · 看槽位状态（核心证据）**

```bash
curl -s -H "Authorization: Bearer <APIKEY>" http://127.0.0.1:8080/slots
```

关注 `is_processing` 与 `next_token.n_decoded` / `n_remain`。

**第 3 步 · 看有没有客户端连着（决定性一步）**

```bash
netstat -ano | grep ":8080"
```

**判读表 —— 这两步一对照就能定性：**

| `/slots` | `netstat` | 结论 |
|---|---|---|
| `is_processing: true` | 有 ESTABLISHED | 正常，有人正在用 |
| `is_processing: true` | **只有 LISTENING / TIME_WAIT** | ★ **被遗弃的生成在空转** |
| `is_processing: false` | 无 | 正常空闲 |

**补充判据 · 进程 CPU 时间**（纯标准库，5 秒窗口内增长接近 5 秒即满载）：用 `ctypes` 调 `GetProcessTimes` 取内核态+用户态时间差。本次事故实测 5 秒内涨 **9.22 s**（≈1.8 核）。

## 六、事故时间线（备查）

| 时间 | 事件 |
|---|---|
| 08:43 | 在 WebUI 里提问，模型陷入死循环 |
| 08:46 | 浏览器被关闭，**但服务端继续生成** |
| 09:15 | 发现 GPU 异常高速运行（82% / 160 W / 70°C） |
| 09:17 | 定位到 `n_decoded = 220,684`，无任何 ESTABLISHED 连接 |
| 09:20 | 重启，GPU 回落至 0% / 39 W / 41°C |
| 09:30 | 加 `-n 32000 --reasoning-budget 8000`，重启生效并验证 |

**代价**：34 分钟满负荷、21.8 万 token 重复内容、约 0.09 kWh。

---

> **一句话**：`llama-server` 的默认值是「无上限 + 不响应断连」，这是个危险组合。
> 加 `-n` 和 `--reasoning-budget` 两个刹车，最坏情况就从「无限」变成「4 分钟」；
> 判断是否空转，看 `/slots` 的 `is_processing` 和 `netstat` 有没有 ESTABLISHED 就够了。
