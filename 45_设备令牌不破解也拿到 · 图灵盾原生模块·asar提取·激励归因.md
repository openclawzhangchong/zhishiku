# 45 · 设备令牌不破解也拿到：官方原生模块就在安装目录里

> **摘要**：客户端用腾讯 T-Sec 图灵盾（TuringShield）给出站请求做「设备证明」（X-Device-Token），外部进程看似无法复现——但官方把**原生模块、渠道号、JS 调用序列**全留在安装目录里。不用破解算法：`node` 加载官方 `turing_sdk.node`，按官方 JS 层的调用序列 `configure(channelId, name, version) → fetchDeviceToken(...)` 就能拿到合法设备令牌。外部进程摇身一变成为「官方客户端设备」，服务端把流量归因到本机设备（激励/积分体系靠它）。

**关键词**：TuringShield, turing_sdk, X-Device-Token, X-Custom-Channel-Id, X-Device-Id-List, qimei36, 设备令牌, 激励归因, channelId, N-API, asar 提取, 每日积分, daily-checkin

---

## 主题 1：设备令牌的完整链路（WorkBuddy 国际版实测）

| 环节 | 内容 |
|---|---|
| **JS 层** | `turing-sdk.js`（asar 内，11KB）：`TuringSdkService` 管理——configure(channelId, productName, version) → fetchDeviceToken({usingCachedMessage, includesOutdatedMessage, includesDeviceInfo}) → 缓存 5 分钟（`DEFAULT_REFRESH_INTERVAL_MS=3e5`），后台线程刷新 |
| **原生层** | `native/turing-sdk/`（asar.unpacked）：`index.cjs`（N-API 包装）+ `turing_sdk.node`（162KB）+ `TuringShieldSDK.dll`（1.5MB，仅导出 `createTSObject`）+ `turing-sdk-variant.json`（`overseas`/`domestic`） |
| **渠道号** | 客户端落盘的产品配置 `~/.workbuddy-ai/cache/acc-product-config-v3-*.json` 里 `config.turingSdk.channelId = 400111`（国际版） |
| **激励头三件套** | `X-Device-Token: <v3:AAA…926字符>`、`X-Custom-Channel-Id: 400111`、`X-Device-Id-List: {"qimei36":"…"}`（qimei36 可选，缺失时客户端省略该头） |
| **验证** | 生成的令牌带三件套发起对话 → 上游 200 接受 |

**可复用教训**：设备证明类 SDK 别急着逆向算法——先看官方有没有把原生模块+配置留在安装目录。**加载官方组件走官方调用序列**，拿到的就是合法令牌（而且官方升级会自动兼容，只要重新指路径）。

**验证状态**：真机实测（token 生成 + 200 对话）

## 主题 2：asar 提取单文件的正确姿势（Python，不依赖 asar 工具）

```python
import json, struct
f = open(asar_path, 'rb')
f.seek(4); header_pickle = struct.unpack('<I', f.read(4))[0]
f.seek(12); json_len = struct.unpack('<I', f.read(4))[0]
header = json.loads(f.read(json_len).decode('utf-8'))
base = 8 + header_pickle          # 数据区起点
# walk header['files'] 树拿到目标文件的 offset/size（offset 是**字符串**，要 int()）
f.seek(base + int(node['offset'])); data = f.read(node['size'])
```

坑：offset/size 在 JSON 里是字符串；`base = 8 + header_pickle` 而不是 `12 + json_len`。

**验证状态**：真机实测（提取 11KB 的 turing-sdk.js 成功）

## 主题 3：集成回 Rust 服务的正解 —— Node sidecar + 后台刷新

| 环节 | 方案 |
|---|---|
| 调用 | `turing_sdk.node` 是 N-API 模块（`napi_register_module_v1`），Rust 无法 libloading 直调（DLL 只导出 `createTSObject` C++ 工厂，vtable 不公开）⇒ **node sidecar**：把生成脚本写临时文件，`node script.cjs <sdkDir> <channelId> <name> <version>`，stdout 拿 token |
| 集成 | **后台刷新**对齐客户端：首次出站先不带激励头，后台线程生成并缓存 5 分钟，之后自动带上——永不阻塞出站热路径 |
| 降级 | node 缺失 / SDK 不存在 → 标记 unsupported，静默不带激励头（行为与旧版一致，零影响） |
| 前提 | 本机装了官方客户端（SDK 文件在安装目录）。没装的话把官方 `native/turing-sdk/` 目录拷到同路径，或评估随包分发（腾讯二进制的版权权衡） |

**验证状态**：真机实测（sidecar exit 0、926 字符 token；Rust 集成后双档冒烟 200）

> 关联：`42_Electron加密凭据与多渠道伪装`（伪装参数按渠道分离——本篇的 channelId 也是按渠道取的）、`39_伪装头对齐与发版兜底`。
