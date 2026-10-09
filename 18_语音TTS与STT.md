# 语音输入（STT）与朗读（TTS）：从"做不到"到"照豆包复刻"的完整过程

> **摘要**：一次完整的语音输入功能落地，最有价值的不是代码，而是**推翻自己上一版结论**的过程：v2 方案断言「按住输入框做不到」，v3 查证后承认**两句话都错了**，改用 `<div>` 伪输入框实现。本文记录这条链路的全部坑位——浏览器录音格式分裂、两个 Whisper 模型的入参格式不同、首次权限必然录空、麦克风不释放、DPI 无关但"朗读 API 整个不存在"的反差——以及一条贯穿始终的原则：**语音是锦上添花，任何时候它坏了，打字这条路必须完好无损。**

**关键词**：MediaRecorder、Whisper、Workers AI、伪输入框、长按判定、getUserMedia、mimeType、base64 音频、speechSynthesis、静默降级、neurons 计费

---

## 一、先认错：v2 说"做不到"，是错的

### 当时的错误论断

> v2 里写：「按住输入框做不到，豆包也不是这么做的」。

**两句话都错了。**

### 推翻过程

1. 查证豆包功能拆解，白纸黑字写着：「语音交互：**长按输入框**或触发语音按钮」
2. 用户给的两张截图印证：空闲态输入框灰色 placeholder「发消息或按住说话...」；按住时变成**黑色居中的**「按住说话」

### 为什么当时会判断错

| 环节 | 对/错 |
|---|---|
| 前提：`textarea` 的长按会被系统输入法接管 | ✅ **这个理由成立** |
| 结论：所以做不到 | ❌ **这一步跳错了** |

**关键差别**：

| 做法 | 结果 |
|---|---|
| 直接用 `<textarea>` 长按 | ❌ 输入法接管，弹键盘、弹全选菜单 |
| **用 `<div>` 伪装成输入框** | ✅ **长按不弹键盘，手势完全自己接管** |

`<div>` 不是输入控件，系统输入法不会碰它。它平时长得跟输入框一模一样，只是**不接受键盘输入**而已。

> **一句话**：不是「按住输入框做不到」，而是「**不能用 textarea 做，得用 div 假装**」。

**✅ 方法论**：当一个结论是「做不到」时，要特别警惕。**"做不到"往往是"用我脑子里那个方案做不到"**，而不是"这件事做不到"。下这种结论前，先问一句：**换一种元素/换一条路，还是做不到吗？**

---

## 二、伪输入框：一个 div，两种用法

```
轻点伪输入框  →  换成真 textarea + 聚焦  →  弹键盘打字
长按伪输入框  →  开始录音，松手发送
```

### 状态机

```
idle ──轻点(180ms内松手)──→ typing
  │                            │
  │                       失焦且为空
  │                            ↓
  └────长按(按满180ms)────→ recording ──松手──→ 发送/填框 ──→ idle
                                 └──上滑──→ 取消 ──→ idle
```

**180ms 是经验值**：人「点一下」通常不到 150ms，真想按住时会自然超过。

**为什么不按下就立刻录音？** 那样每次轻点打字都会先启动一次录音，还会**触发麦克风权限弹窗**。先判意图再录音，干净。

### 核心代码骨架

```js
const LONG_PRESS_MS = 180;

// iOS 只认 mp4，安卓/桌面走 webm。必须动态选，写死一个必挂一半设备
const REC_MIMES = ['audio/webm;codecs=opus', 'audio/webm', 'audio/mp4', 'audio/ogg'];
function pickMime() {
  if (!window.MediaRecorder?.isTypeSupported) return '';
  return REC_MIMES.find((m) => MediaRecorder.isTypeSupported(m)) || '';
}

// 语音输入到底可不可用：后台总闸开着 + 浏览器支持录音
function micAvailable() {
  return voiceInputOn
    && typeof MediaRecorder !== 'undefined'
    && !!navigator.mediaDevices?.getUserMedia;
}

// 关掉/不支持 = 完全退回现在的样子（永远显示真 textarea）
function applyVoiceInput() {
  const pad = $('#micPad'), ta = $('#input');
  if (micAvailable()) { pad.hidden = false; ta.hidden = true; paintPad(); }
  else { pad.hidden = true; ta.hidden = false; }
}
```

### 配套 CSS（两行是关键）

```css
#micPad {
  touch-action: none;                 /* 关键：长按时禁止页面滚动 */
  user-select: none; -webkit-user-select: none;
}
#micPad.rec { color: var(--tx); font-weight: 500; text-align: center; }   /* 录音态居中 */
```

**注意**：空闲态**左对齐**，录音态**居中**——跟豆包一致。

---

## 三、★ 服务端：两个 Whisper 模型的入参格式不一样

这是**实测踩到**的坑，报错原文：

```
Type mismatch of '/audio', 'string' not in 'array','binary'
```

| 模型 | `audio` 参数格式 | 其他 |
|---|---|---|
| `@cf/openai/whisper` | **0–255 整数数组** | — |
| `@cf/openai/whisper-large-v3-turbo` | **base64 字符串** | ⚠️ `language` **默认是 `'en'`**，不显式给 `'zh'` 中文会走英文路径 |

```js
const MODELS = {
  turbo: '@cf/openai/whisper-large-v3-turbo',   // 46.63 neurons/分钟
  base:  '@cf/openai/whisper',                   // 41.14 neurons/分钟
};

const out = key === 'base'
  ? await env.AI.run(MODELS.base,  { audio: [...bytes] })
  : await env.AI.run(MODELS.turbo, { audio: bytesToBase64(bytes), language: 'zh' });
```

**✅ 教训**：**同一个"Whisper"名字下的不同模型，API 契约可能完全不同。** 换模型不等于换个字符串——要重新验证入参格式。

---

## 四、必须处理的十个坑

| 坑 | 现象 | 处理 |
|---|---|---|
| **首次权限** | 第一次长按，权限框弹出时手指已松 → **第一次录音必然是空的** | 首次长按只用来申请权限，成功后提示「已开启麦克风，再按一次说话」，**不报错** |
| **格式分裂（客户端）** | iOS 只认 `audio/mp4`，安卓/桌面是 `audio/webm` | `pickMime()` 动态选 |
| **格式分裂（服务端）** | 两个 whisper 模型入参格式不同 | 按模型分支（**实测踩过**） |
| **长按选中文字** | 长按伪输入框弹出选择菜单 | `user-select: none` + `preventDefault` |
| **长按时页面滚动** | 手机长按，页面跟着滚 | `touch-action: none` |
| **手指滑出** | 松手时 `pointerup` 不触发，一直录 | 绑 `pointercancel` 兜底 |
| **麦克风没释放** | 录完后手机一直显示"正在录音" | `recStream?.getTracks().forEach(t => t.stop())` |
| **iOS 14.5 以下** | `MediaRecorder` 不存在 | `micAvailable()` 返回 false，退回普通输入框 |
| **轻点被误判成长按** | 想打字结果录了音 | 180ms 阈值；且录音 < 0.5 秒不发 |
| **长按被误判成轻点** | 想说话结果弹了键盘 | 180ms 偏短，可按实测调到 200–250ms |

**误触判据**：`blob.size < 1200` 视为误触不发送（5.5 秒正常录音约 75 KB）。

---

## 五、★ 一个必须记住的反差：同一个浏览器，录音能用、朗读不能用

TTS 那次实测的结论是「**小米浏览器整个 `speechSynthesis` 不存在**」，但**同一个浏览器的录音 API 完全正常**。

> **「小米浏览器不支持语音」是错的**——不支持的是**朗读**，录音是好的。

**✅ 方法论**：**永远不要用"这个浏览器不支持语音"这种笼统结论**。要具体到 API 名：
- `MediaRecorder`（录音）
- `navigator.mediaDevices.getUserMedia`（拿麦克风）
- `SpeechRecognition`（语音转文字，Chrome 走 Google 服务器，**国内不通**）
- `speechSynthesis`（朗读）

四个是四件事，任何一个缺失都不能推及其他。

### 为什么不用浏览器自带 SpeechRecognition

| 方案 | 为什么不用 |
|---|---|
| 浏览器自带 `SpeechRecognition` | Chrome 走 Google 服务器，国内不通；很多国产浏览器整个 API 不存在 |
| 浏览器本地跑 Whisper | 要下 40–250MB 模型，手机跑得慢，费流量费电 |
| **Workers AI Whisper** | ✅ 采用：服务端识别，零客户端体积 |

---

## 六、静默降级：语音坏了，打字必须还在

```js
} catch (e) {
  toast(e.message);                 // 一句人话，不弹技术错误
}
```

**核心原则**：

> **语音输入是锦上添花。任何时候它坏了，打字这条路必须完好无损。**

具体的三条保障：
1. 后台总闸关掉 / 浏览器不支持 → `#micPad` 永远不出现，`#input` 一直显示，**表现和没有这个功能时一模一样，零影响**
2. 服务端也要判总闸（`getConfig(...) === '0'` → 403），**就算前端还留着旧页面**也拒
3. 识别失败给一句人话（「没听清，再说一次」），不暴露 HTTP 状态码、模型名、错误栈

---

## 七、成本（可复制的算法）

| 项 | 值 |
|---|---|
| 费率（turbo） | 46.63 neurons / 音频分钟 |
| 费率（base） | 41.14 neurons / 音频分钟（省 12%） |
| 免费额度 | 10,000 neurons/天 ≈ **214 分钟 ≈ 3.6 小时语音** |
| 折合句数 | 10 秒/句 → 约 **1,280 句/天** |
| 超额 | $0.011 / 1000 neurons → 1 小时音频约 **¥0.2** |

**⚠️ 额度是整个 Cloudflare 账号共用的**（所有用户合计）。

**体积参考**：5.5 秒 = 75 KB → 60 秒上限约 820 KB。服务端闸门设 `MAX_BYTES = 2 * 1024 * 1024`，留足余量。

---

## 八、实测数字（来自一次性测试站）

```
服务端（TTS 生成的中文 MP3 打接口）：
POST /api/stt  content-type: audio/mpeg  30816 字节
→ ok:true  audioKind:base64  detectedLang:zh   aiMs: 2424ms
  text: "你好,这是一段语音识别的测试,请把这句话识别出来。"   ← 一字不差

手机端（小米浏览器 / Android 16）：
audio/webm;codecs=opus = true   audio/webm = true   audio/mp4 = true
audio/ogg / audio/wav / audio/mpeg = false
录音 5.50s → 75094 字节 → HTTP 200  往返 1352ms（其中 AI 937ms）
text: "在吗在吗这是一个测试的语音测试能不能语音输入"   ← 一字不差
```

| 验证项 | 结果 |
|---|---|
| 小米浏览器能不能录音 | ✅ 能 |
| 录出来什么格式 | `audio/webm;codecs=opus` |
| whisper 认不认 | ✅ **直接认，不需要转码** |
| 中文准确度 | ✅ 一字不差 |
| 松手到出字延迟 | ✅ 约 **1.35 秒** |
| 微信内置浏览器 | ✅ 可用 |
| 首次权限耗时 | 2.95 秒（第二次会快很多） |

---

## 九、失败清单

| # | 失败模式 | 表现 | 防法 |
|---|---|---|---|
| 1 | 用 textarea 做长按 | 输入法接管，弹键盘 | 用 `<div>` 伪输入框 |
| 2 | 写死 mimeType | iOS 或安卓一边挂 | `pickMime()` 动态探测 |
| 3 | 假设两个 whisper 同契约 | 502 `Type mismatch` | 按模型分支 + 实测 |
| 4 | turbo 不给 `language:'zh'` | 中文走英文路径，效果差 | 显式传 |
| 5 | 没处理首次权限 | 第一次录音必空 | 首次只申请权限 + 提示 |
| 6 | 不关音轨 | 手机一直显示录音中 | `getTracks().forEach(stop)` |
| 7 | 没绑 `pointercancel` | 手指滑出后一直录 | 绑兜底事件 |
| 8 | 前端判了后端没判 | 旧页面绕过总闸 | 服务端二次校验 |
| 9 | 弹技术错误给用户 | 用户看不懂且恐慌 | 一句人话 |
| 10 | 笼统说"浏览器不支持语音" | 误杀可用能力 | 具体到 API 名 |

---

## 十、一句话总结

> **语音功能的所有难点都在"降级"上：格式要动态选、权限要容错、麦克风要释放、服务端要二次校验、失败要说人话。**
> 而最贵的一条教训是——**别急着说"做不到"**。那个结论往往只说明你想到的那条路走不通。
