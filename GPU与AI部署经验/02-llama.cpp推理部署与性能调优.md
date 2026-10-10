# 02 · llama.cpp 推理部署与性能调优（V100 / Windows）

> **摘要**：记录在本机 Tesla V100-32GB 上用官方 llama.cpp（build 10903，CUDA 12.4）部署 GGUF 大模型的全过程经验：从模型架构取证、KV 显存精确计算、性能基线标定，到 MTP 自投机解码（最高 154 t/s）、bat 三件套交付与官方 WebUI 行为。重点标出“已实测排除、勿重复试”的参数，避免在无用调优上浪费时间。
>
> **关键词**：llama.cpp；GGUF；KV Cache；显存计算；MTP 投机解码；sm_70；CUDA 12.4；性能基线；WebUI；bat 三件套；Qwen3；gpt-oss

---

## 一、踩过的坑（吃过哪些亏）

1. **凭模型名猜架构导致算错显存**：Qwen3.8 与 Qwen3.6 架构完全不同（qwen35 vs qwen35moe），必须解析 GGUF 元数据。
2. **用“总层数”算 KV 高估 4 倍**：混合注意力模型只有部分层是全注意力，必须用“全注意力层数”。
3. **升级覆盖丢 CUDA 运行时**：官方 Windows 主包不含 `cublas64_12/cublasLt64_12/cudart64_12.dll`，覆盖升级未回填 → `0xc0000135`。
4. **新包丢 sm_70 PTX**：CUDA ≥12.3 编译的包若无 sm_70 SASS，旧驱动必失败 → 换包前必须解剖 `ggml-cuda.dll` 确认仍含 sm_70。
5. **首次运行慢误判为故障**：实际是 PTX JIT 预热（`%APPDATA%\NVIDIA\ComputeCache`），首跑 1.19 t/s → 次跑 101 t/s，慢在 `listening on` 之后属正常。
6. **报速度不注温度+场景**：greedy 短问答 119 t/s，但 WebUI 默认 temp=1.0 写作文只有 80~90 t/s，差异被当成故障。
7. **reasoning-budget 与 max_tokens 共享额度**：budget=2048 + max_tokens=900 时，复杂题把 900 额度全耗在思考上、答案 0 字。
8. **bench 显存不够开两个实例**：23+23 > 32GB ⇒ 新进程直接退出且日志无显存报错（只表现为“配了 cmd 却无 RESULT”）。
9. **官方 WebUI gzip 内嵌**：不带 `Accept-Encoding: gzip` 访问 `/` 得 `415 Error: gzip is not supported`（这恰好是“官方界面已生效”的判据）。`/bundle.xxx.js` 返回 401 而非 404（鉴权中间件先于静态路由）。

## 二、怎样才能成功（成功做法）

- **第一步永远是解析 GGUF 元数据**：工具 `_ggufdump2.py`；GGUF 架构键带前缀（正则不能加行首锚点）；跳大数组必须真正 seek 掉（否则后续全读偏）；`nextn_predict_layers >= 1` ⇒ 支持内置 MTP。
- **KV/token 公式（勿算错）**：
  `KV/token(f16) = 全注意力层数 × kv_heads × head_dim × 2 × 2字节`；`q8_0` 为其一半。
- **MTP 是唯一有效提速手段**：`--spec-type draft-mtp --spec-draft-n-max 3`（3 也是官方默认）；得益 qwen35moe 86→119 t/s，高可预测内容峰值 154 t/s。
- **写速度必须注明温度+场景**；写长文想提速让客户端把 temperature 调到 0.6~0.8（+7~13%），不必改服务端 bat。
- **每模型交付三件套**：① 交互式菜单启动 bat（档位/无效输入重试/8080 占用检测）② 停止 bat（检测→确认→强杀→复验，只按映像名精确匹配）③ 面向初学者的 readme（含显存推导、故障排查、实际速度口径）。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **qwen35moe 反直觉**：f16 vs q8_0 深度测试（0/64K/128K）= 88.4/77.7/67.6 vs 86.6/59.4/49.0 → **f16 全胜，越深越赢**，与 Qwen3.8 相反，不能套用。两档都用 f16。
- **MTP n_max 随模型规模变**：35B MoE 用 3；小稠密 9B 用 2 更优（每多验证一个草稿 token 就多算一次前向，小模型验证成本更快抵消收益）。≥4 反而变慢。
- **采样削弱 MTP**：temp=1.0 接受率 80%→64%。要跑满速度用 `--temp 0`。对比 n_max 必须固定 `seed`。
- **V100 速度天花板**：空载 ~90.6 t/s，深度衰减 64K→77.7、128K→67.6、256K≈57；SXM2=98.8；RTX 3090=100~120。本机比 SXM2 低 10% 是时钟差（1380 vs 1530 MHz），参数补不回。GPU util 仅 70~83%、功耗 121~145W ⇒ 瓶颈是 MoE 小矩阵运算效率。
- **已实测排除、勿重复试**：`-b/-ub` 放大、`FORCE_MMQ`、`FORCE_CUBLAS`、`CUDA_GRAPH_OPT`、加线程 → 零变化；`DISABLE_FUSION=1` **慢 21%**；`-fa 0` 仅 +0.8%。`llama-bench` 的 `-fa` 写 **`1`**（写 `on` 静默零输出）。

## 四、具体怎么实现（实现方式）

**1. 安装包获取与覆盖升级**
- 主包 `llama-b10903-bin-win-cuda-12.4-x64.zip`：`github.com/releases/download` 直连被 reset → 走 API 资产端点 `GET /releases/assets/<id>` + `Accept: application/octet-stream`，302 跳 `objects.githubusercontent.com`，加 `--location-trusted`。
- 覆盖脚本 `_swap_bin.py`（dry-run → `--apply`）：停服 → 旧 `*.exe/*.dll` 移 `_bin-backup-<时间戳>\`（同盘秒级回滚）→ 复制白名单 `.exe/.dll`+LICENSE（绝不碰 models/、*.bat、*.md、webui/）→ 回填 3 个 CUDA 运行时 DLL。

**2. 性能基线（驱动 551.61，build 10903）**

| 模型 | 档位 | KV | 显存峰值 | 生成 t/s |
|---|---|---|---|---|
| Qwen3.8-27B | 64K/128K/256K | f16/f16/q8_0 | 20276/24436/26060 MiB | 33.05/32.98/32.10 |
| Qwen3.6-35B-A3B | 128K/256K | f16/f16 | 19829/22517 MiB | 85.2/85.9 |
| gpt-oss-20b | 32K/128K | — | 20.4/22.7 GB | 294~297 |

跨版本/跨配置比性能必须同脚本同请求；带采样的 API 口径比裸跑低 10%+，不可直接相减。

**3. V100 启动 bat 硬规则**
- 必须 **GBK + CRLF**；必加 `set GGML_CUDA_PDL=0`（V100 上 PDL 误报 `invalid device function`）、`--jinja`（Qwen 系否则多轮错乱）、`--api-key`；**不要加 `--path webui`**。
- 采样：Qwen 系 `--temp 1.0 --top-p 0.95 --top-k 20 --min-p 0`；gpt-oss 走 `--chat-template-kwargs "{\"reasoning_effort\":\"low\"}"` 且必须 `-fa on`。
- 生成器 `_gen_qwen3827b.py` / `_gen_qwen36moe.py` / `_gen_qwen36mtp.py` / `_gen_tunnel.py`（Python 写 GBK+CRLF，加模型＝改 `MODELS` 重跑生成器）。交付前必须实测并回填 readme。

**4. 思考深度**：Qwen 系走 `--reasoning auto` + `--reasoning-budget -1` 会暗中大量思考拖慢首字；菜单四档 `on/-1 / 2048(推荐) / 512 / off`。关掉/调低思考不会让正文逐字变快，但能消除“先暗中思考好几秒才出字”的等待感。
