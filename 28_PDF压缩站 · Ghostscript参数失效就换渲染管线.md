# PDF 压缩站 · Ghostscript 参数失效就换渲染管线

> **摘要**：2026-10-24 单日从零做出一个纯前端 PDF 批量压缩站（zcagent/智策张翀），部署 WorkBuddy 站点托管 + 开源到 GitHub。核心结论三条：①**别在本地反复起 server 测，直接上线 + 页内日志面板，看日志改代码**——这是用户明确纠正过的元教训，也是最省时间的方式；②**Ghostscript 10.08 WASM 的 `DownsampleColorImages`/`JPEGQ` 参数全部失效**（6 种写法输出恒定），照片型必须换 pdf.js 渲染重采样管线，文字型才用 gs 保文本层；③**worker 里的静态 `import` 失败是静默的**，必须全改动态 import + 哨兵消息，否则永远只看到「Worker 异常：未知」。全程踩了 20 个坑，其中 6 个是"看着像 A 其实是 B"。
>
> **关键词**：PDF 压缩、Ghostscript WASM、pdf.js 渲染重采样、Web Worker ESM、OffscreenCanvas、EdgeOne CDN 缓存、CDP 自动化测试、GitHub API 推送、双管线路由
>
> **触发场景**：要做浏览器端 PDF 处理（压缩/转图/合并）；要在 worker 里加载 wasm 或 ESM 库；静态托管部署后线上跑旧代码；agent-browser 挂掉需要替代自动化方案。
>
> **整理日期**：2026-10-24
>
> **产物**：https://pdf-compress-zcagent.app.workbuddy.host/ ｜ https://github.com/openclawzhangchong/pdf-compress-zcagent

---

## 一、元教训：不要本地反复起 server，直接上线测

**这是本篇最重要的一条，排在所有技术坑前面。**

我一开始在本地起 `python -m http.server`，用 agent-browser 反复测。结果：
- server 后台进程反复被杀（`(... &)` 脱离的进程被回收），每次测之前先修环境；
- agent-browser 的 daemon 后来彻底起不来（`open` 即 SIGTERM，`doctor` 也超时）；
- 花了约 40 分钟在**环境**上，而不是在**功能**上。

用户原话纠正：

> 「你为什么一定要在本地测试呢？你直接给它上到 WorkBuddy 的站点上面去，然后把这个日志进行输出嘛……测试了有哪些日志的话，我们就直接修代码不就好了？本地一直开就存在这种问题啊。」

**正确姿势（后来一次跑通）**：

```
改代码 → 改资源版本号 → 部署（一条命令）→ 线上打开 ?debug=1 → 读日志 → 定位 → 改代码 → 再部署
```

一轮 2-3 分钟，且**测的就是真实环境**（CDN、MIME、sharecode 校验、CORS 全都在）。

**前提是做好两件事**：

1. **页内日志面板**（日志必须渲染进 DOM，不能只 `console.log`）——这样自动化能直接读到，用户也能截图给你：

```js
var DEBUG = /[?&]debug=1/.test(location.search);
var logLines = [];
function dlog(msg) {
  var d = new Date(), pad = x => (x < 10 ? '0' : '') + x;
  logLines.push(pad(d.getHours()) + ':' + pad(d.getMinutes()) + ':' + pad(d.getSeconds()) + ' ' + msg);
  var el = document.getElementById('debugLog');
  if (el) { el.textContent = logLines.join('\n'); el.scrollTop = el.scrollHeight; }
}
```

2. **worker 的日志必须 `postMessage` 回主线程**——worker 里的 `console.log` 在主线程面板里是看不到的。

```js
function log(msg) { self.postMessage({ type: 'log', msg: String(msg) }); }
```

---

## 二、选型：pdf24 类工具的开源参考

| 项目 | Stars | 适合 |
|---|---|---|
| Stirling-PDF | 94.8k | 全功能自托管（Java/Docker），60+ 工具，**但要服务器** |
| divyesh1099/compressPDF | — | Flask + PyMuPDF + pikepdf + Ghostscript，专做批量压缩 |
| afeiship/ghostscript-pdf-compress.wasm | — | 浏览器内 Ghostscript 压缩的最小 demo |
| PDF-Verse / PDFBuddy / ilovepdf-clone | <300 | pdf-lib 纯前端克隆，**压缩能力很弱** |

**关键判断**：pdf-lib 只能做对象流/字体子集压缩，扫描件基本压不动。真压缩必须走「图像重编码 + Ghostscript」或「渲染重采样 + JPEG」。纯前端要做到可控压缩率，**只能自己搭渲染管线**。

---

## 三、双管线路由（本项目的核心设计）

```
PDF 进来 → pdf.js 探测每页文本量
   ├─ 平均每页 < 200 字  → 照片型 → pdf.js 渲染 + OffscreenCanvas 重采样 + JPEG 重建
   └─ 平均每页 ≥ 200 字  → 文字型 → Ghostscript pdfwrite 强制重编码（保文本层）
```

**为什么必须分两条**：
- 照片型走渲染，可以精细控制 dpi 和 JPEG 质量（gs 的参数在这个 wasm 构建里失效，见坑 3）；
- 文字型走渲染会把文字变成图片，**文本层丢失、不可搜索**，是灾难性降级；
- Ghostscript 17MB wasm **按需加载**（只有文字型才拉），照片型首屏只需 1.7MB 的 pdf.js。

---

## 四、20 个坑（按真实踩坑顺序）

### 坑 1 · Ghostscript 10.08 WASM 的压缩参数全部失效

**症状**：`compressPdf(input, {preset:'screen'|'ebook'|'printer'})` 三档输出**完全一样大**，5.16MB → 4.92MB（省 0.1%）。

**排查过程**（纯耗时，记下来让别人别重复）：
1. 怀疑 preset 没传 → 读 `operations.js` 源码，确认 `-dPDFSETTINGS=/${preset}` 确实拼进去了；
2. 怀疑降采样被 JPEG 透传绕过 → 加 `-dPassThroughJPEGImages=false`，**有效**（省 29.9%）；
3. 但 `-dJPEGQ=30/-dColorImageResolution=100/-dDownsampleColorImages=true` 各种组合输出**恒等**；
4. 怀疑新版解释器 → 加 `-dNEWPDF=false`，输出**还是恒等**；
5. 试 PostScript 写法 `-c "<</ColorImageResolution 72 /JPEGQ 40>> setpagedevice"`，**还是恒等**。

**结论**：这个 wasm 构建里只有 `PassThroughJPEGImages=false`（强制重编码，质量写死默认值）生效，其余图像参数全部被忽略。

**解法**：照片型不用 gs，改走 pdf.js 渲染重采样（参数完全可控）；gs 只留给文字型做强制重编码。

---

### 坑 2 · 真正原因是文件结构，不是引擎（"看着像 A 其实是 B"）

继续挖才发现：那个 PDF 的 **MediaBox 就是像素值**（2328×3264 **pt**），是手机扫描 App 的产物——图像在页面上是 **1:1 显示，有效 dpi 只有 72**。

**任何 ≥72dpi 的降采样目标都不会触发降采样**，因为已经没有降采样空间。压缩空间只在 JPEG 质量维度。

**顺序教训**：遇到"压缩没效果"，**先看文件结构再看引擎**：

```bash
# 看图像滤镜类型和数量
grep -o "/DCTDecode\|/FlateDecode\|/JBIG2Decode\|/CCITTFaxDecode" file.pdf | sort | uniq -c
# 看图像像素尺寸
grep -o "/Width [0-9]*\|/Height [0-9]*" file.pdf | head
# 看页面尺寸(对象流压缩的 PDF 要先用 pdf.js/pdf-lib 打开)
```

用 pdf.js 拿真实页面尺寸：`page.getViewport({scale:1}).width` → 2328pt。**图像像素 ÷ (页面 pt / 72) = 有效 dpi**。

---

### 坑 3 · `Uint8Array.toString('latin1')` 不是解码，是 `[object Uint8Array]`

**症状**：压缩输出被自己的校验函数判为"非法 PDF"，但文件其实是好的（`%PDF-1.5` 开头，`%%EOF` 结尾）。

**根因**：Node 里 `Buffer.prototype.toString(encoding)` 支持编码，`Uint8Array.prototype.toString()` 忽略参数，返回 `"[object Uint8Array]"`。

**解法**：

```js
function isValidPdf(buf) {
  const b = Buffer.from(buf);          // 必须先转 Buffer
  return b.slice(0, 8).toString('latin1').startsWith('%PDF-')
      && b.slice(-1024).toString('latin1').includes('%%EOF');
}
```

**教训**：**校验函数本身也要被校验**。第一次拿到"3 个全失败"时，先怀疑校验器而不是产线。

---

### 坑 4 · worker 里静态 `import` 失败是静默的（最坑）

**症状**：线上点击压缩，只报 `Worker 异常：未知`，`e.message`/`e.filename`/`e.lineno` 全空。

**根因**：`type:'module'` 的 worker 里，顶层静态 `import` 失败时浏览器不给主线程任何细节——这是模块加载错误的静默特性。

**解法**：**依赖全部改动态 import，并加哨兵消息**。

```js
// worker 顶部
var DEP = {};
async function loadPdfLibs() {
  if (DEP.pdfjs) return DEP;
  log('加载 pdf.js(1.7MB)...');
  var pdfjs = await import('../vendor/pdfjs/pdf.min.mjs');     // 失败可 catch
  log('pdf.js OK v' + pdfjs.version);
  pdfjs.GlobalWorkerOptions.workerSrc =
    new URL('../vendor/pdfjs/pdf.worker.min.mjs', import.meta.url).href;
  var pl = await import('../vendor/pdf-lib/pdf-lib.esm.min.js');
  DEP.pdfjs = pdfjs; DEP.PDFDocument = pl.PDFDocument;
  return DEP;
}
// 脚本加载成功的哨兵(主线程据此判断模块解析是否通过)
self.postMessage({ type: 'log', msg: 'worker 脚本已加载(模块解析成功)' });
```

主线程侧同时把 onerror 的细节打全：

```js
w.onerror = function (e) {
  dlog('❌ Worker 异常: message=' + (e.message || '(空)') +
       ' file=' + (e.filename || '(空)') +
       ' line=' + e.lineno + ':' + e.colno +
       (e.error ? ' err=' + e.error.message : ' err=(无)'));
};
```

---

### 坑 5 · pdf.js 6.x 的 `PDFDocumentProxy` 没有 `destroy()`

**症状**：`doc.destroy is not a function`。

**解法**：6.4.299 的文档代理只有 `cleanup()` / `startCleanup()`；`destroy()` 在 `PDFDocumentLoadingTask` 上。

```js
try { doc.cleanup(); } catch (e) { log('cleanup 忽略: ' + e.message); }
```

---

### 坑 6 · `doc.getPage(i)` 漏 await → 所有 PDF 被误判照片型（最隐蔽）

**症状**：日志每页都打 `文本读取失败: doc.getPage(...).getTextContent is not a function`，然后「平均文本 0 字 → 照片型」。

**后果**：**纯文字课件会被当成照片渲染成图片，文本层全丢**。这是最危险的一个 bug，因为不报错、不阻断，只是悄悄降级。

**根因**：

```js
var tc = await doc.getPage(i).getTextContent();   // ❌ getPage 返回 Promise
var pg = await doc.getPage(i);                    // ✅
var tc = await pg.getTextContent();
```

**教训**：被 `try/catch` 包住的探测型代码，**失败会静默变成"默认值"**，必须把 catch 里的日志也当成一级信号看（我是从日志里那一行 `文本读取失败` 才发现的）。

---

### 坑 7 · 档位键不一致 → 三档输出完全一样

**症状**：low/mid/high 三档的每页 JPEG 大小一字不差（387/494/549/554/603/545/355 KB）。

**根因**：主线程传 `PRESETS[state.preset].gs`（`screen`/`ebook`/`printer`），worker 的照片档位键是 `low`/`mid`/`high`：

```js
var t = PHOTO_TIERS[tierKey] || PHOTO_TIERS.mid;   // 永远 fallback 到 mid
```

**教训**：**"档位没区别"= 参数没传到**。带 fallback 默认值的参数解析，会吃掉所有拼写错误。日志里打出实际生效的参数：

```js
log('档位 ' + tierKey + ': dpi=' + t.dpi + ' quality=' + t.quality + ' scale=' + t.ptPxScale);
```

---

### 坑 8 · EdgeOne CDN 缓存：部署成功但线上跑旧代码

**症状**：部署工具返回成功，但线上行为还是旧的；`curl` 拿到的和新版特征对不上，而浏览器又跑的是新版。

**证据链**：

```
curl -I ... → EO-Cache-Status: HIT, Age: 17
加 ?_=timestamp 随机参数 → 拿到的是新内容
```

**解法**：业务脚本一律带版本号，改代码必改版本号（两处必须一致）：

```html
<link rel="stylesheet" href="css/style.css?v=1024i">
<script src="js/app.js?v=1024i" defer></script>
```

```js
var ASSET_VER = '1024i';   // 改代码必改这里
new Worker('js/compress.worker.js?v=' + ASSET_VER + '&' + location.search.replace(/^\?/, ''), { type: 'module' });
```

**教训**：`vendor/` 的第三方库内容稳定，不加版本号反而省流量；**只有自己会改的业务文件才需要破缓存**。

---

### 坑 9 · 分享链接的 sharecode 校验会拦子资源

**症状**：线上 worker 加载失败，但页面本身正常（`<script src>` 能加载）。

**证据**：

```bash
curl ".../js/compress.worker.js"                    # 403 + HTML 登录页(2988B)
curl ".../js/compress.worker.js?sharecode=xxx"      # 200 + text/javascript(6170B)
```

**解法**：worker URL 拼上 `location.search`：

```js
new Worker('js/compress.worker.js?v=' + ASSET_VER + '&' + location.search.replace(/^\?/, ''), ...);
```

---

### 坑 10 · `hidden` 属性被 `display:flex` 覆盖

**症状**：非调试模式下，调试日志黑框一直挂在页面右下角（用户一打开就看到）。

**根因**：`.debug-panel { display: flex }` 是作者样式，优先级高于 `hidden` 属性对应的 UA 样式 `display:none`。

**解法**：

```css
.debug-panel[hidden] { display: none !important; }
```

**教训**：**凡是设置了 `display` 又靠 `hidden` 属性控制的元素，都有这个坑**。反过来讲，`hidden` input 找不到元素（自动化工具无法 upload）时，也改成 CSS 屏幕外定位：

```css
.off-screen { position: absolute; left: -9999px; width: 0; height: 0; opacity: 0; }
```

---

### 坑 11 · agent-browser 挂了：daemon 起不来

**症状**：`agent-browser open <any-url>` 直接 SIGTERM 退出，`doctor` 60 秒超时无输出，`close` 正常，`--version` 正常。残留进程排查无 chrome 进程，磁盘正常。

**结论**：不必死磕。**改用系统浏览器无头 + CDP（Chrome DevTools Protocol）手写客户端**，Node 22+ 自带 WebSocket，零依赖：

```js
import { spawn } from 'node:child_process';
const edge = spawn('C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe', [
  '--headless=new', '--disable-gpu', '--no-sandbox',
  '--remote-debugging-port=9300',
  '--user-data-dir=C:/.../Temp/edge-cdp-' + Date.now(),
  '--window-size=1280,1100', 'about:blank',
], { stdio: 'ignore' });

// 等 /json 就绪 → 拿 webSocketDebuggerUrl → new WebSocket(wsUrl)
// send('Runtime.evaluate', {expression}) 执行 JS，send('Page.captureScreenshot') 截图
```

**CDP 三个救命命令**：

| 命令 | 用途 |
|---|---|
| `DOM.setFileInputFiles {files:[绝对路径], objectId}` | **塞本地文件进 input**（替代 upload，不再有上下文丢失问题），自动触发 change |
| `Browser.setDownloadBehavior {behavior:'allow', downloadPath}` | 允许下载落盘 |
| `Emulation.setDeviceMetricsOverride` + `Page.captureScreenshot {captureBeyondViewport}` | 高清截图 |

---

### 坑 12 · agent-browser 的 upload 会重置页面上下文

**症状**：`upload` 成功后，下一条 `eval`/`click` 报 `Cannot read properties of null`，`location.href` 变成 `about:blank`；但 `snapshot` 还能读到完整 DOM。

**规律**：open 之后第一条命令有效，upload 之后上下文必掉。

**绕法**（三选一，按可靠性排序）：
1. **CDP `DOM.setFileInputFiles`**（最优，见坑 11）；
2. 用 iframe 托管页把注入逻辑写成页面脚本（`fetch` 测试文件 → `new File` → `DataTransfer` → `input.files = dt.files` → `dispatchEvent(change)`），一条 `open` 命令搞定全部；
3. 命令行内联 base64 塞文件——**Windows 会踩 `Argument list too long`（32KB 上限）**，41KB 的 base64 直接失败。

---

### 坑 13 · 示例/演示素材太"干净"，压缩率看不出来

**症状**：截图用的示例 PDF 只有 300KB，压缩后 336KB（反而变大），走"无压缩空间"兜底——**演示效果为零**。

**根因**：canvas 画的文档（渐变 + 文字）JPEG 压缩效率极高，原文件本身就小。

**解法**：让示例具备**真实拍照件的特征**：
- 尺寸 2400×3400（约 800 万像素，手机典型）
- 26000 个纸张纤维噪点（半透明，2-4px）
- 四周暗角 `createRadialGradient`
- 两页不同内容（真实文档不会页页相同）
- `toDataURL('image/jpeg', 0.95)`（接近手机直出）

改完：**1.65 MB → 176 KB，省 90%**，截图才有说服力。

**教训**：做演示素材要**还原真实数据的"脏"**，否则压缩率、体积、渲染时间全都是假数据。

---

### 坑 14 · 空 GitHub 仓库不能建 blob（409）

**症状**：`POST /repos/{o}/{r}/git/blobs` 返回 `409 Git Repository is empty`。

**解法**：空仓库必须先有一次初始提交。用 Contents API 写第一个文件：

```python
api(f'{RB}/contents/README.md', {'message': 'init', 'content': base64(readme)})
```

之后四步法（blob→tree→commit→ref）正常工作。

---

### 坑 15 · 空仓库 `PUT /contents` 指定 `branch: main` 会 404

**症状**：`PUT /repos/{o}/{r}/contents/README.md` 带 `"branch": "main"` → `404 Not Found`（不是 422，很有迷惑性）。

**根因**：空仓库**还没有 main 分支**，指定一个不存在的 branch 就是 404。

**解法**：**不要传 branch 参数**，GitHub 会自动创建默认分支：

```python
api(f'{RB}/contents/README.md', {'message': 'init', 'content': base64_text})   # 201
```

---

### 坑 16 · 17MB wasm 的 base64 上传可行，但要给足超时

`wghostscript.wasm` 16.8MB → base64 约 22.4MB，`POST /git/blobs` 耗时 **12.7 秒**，一次成功。urllib 默认超时要调到 300s，否则大文件必失败。

**本地 sha1 前检**（省一次往返，也能立刻发现编码错）：

```python
local_sha = hashlib.sha1(b'blob ' + str(len(raw)).encode() + b'\x00' + raw).hexdigest()
blob = api(f'{RB}/git/blobs', {'content': base64.b64encode(raw).decode(), 'encoding': 'base64'})
assert blob['sha'] == local_sha
```

---

### 坑 17 · headless 浏览器的 blob 下载不落盘

`a.download` + `URL.createObjectURL(blob)` 在 headless 下即使设置了 `Browser.setDownloadBehavior`，下载目录仍为空。

**解法**：不靠文件，靠**页内日志**验证——打包完成时打日志（体积 + 文件名），这才是可自动读取的证据：

```
10:25:50 开始打包 ZIP,共 2 个文件
10:25:50 ZIP 打包完成 2.65MB,文件名 compressed_20261024_1025.zip
```

---

### 坑 18 · 压缩率预期要按文件类型分开看

同类工具宣传"压缩 90%"，但**同一套参数在不同文件上差 5 倍**：

| 文件类型 | 特征 | 中档压缩率 |
|---|---|---|
| 高分辨率拍照件（2400×3400 放在 A4 页） | 有效 290dpi | **90%** |
| 扫描 App 产物（MediaBox=像素值，72dpi） | 分辨率已到底 | **46%** |
| 手机拍照作业（4.92MB/7页，q90） | 72dpi + 高质量 | 46% |
| 纯文字 PDF | 走 gs 重编码 | 30% |

**教训**：跟用户沟通压缩率，**必须先说清文件属于哪一类**，否则"为什么别人的能压 90%"会变成信任问题。

---

### 坑 19 · `pdf-lib` 不能替换 PDF 内已有图像

想做"只重编码内嵌 JPEG、不重建页面"，pdf-lib **没有替换图像对象的 API**。可行路径只能整页重建（渲染重采样），代价是文字型 PDF 会丢文本层——这也是双管线必须分流的根本原因。

---

### 坑 20 · `doc.cleanup()` 不是万能的清场

`cleanup()` 只是释放缓存的页面资源，**wasm 实例和 17MB 内存不会释放**。批量处理很多文件时，gs 实例要复用（`gsInstance` 单例），不要每个文件重新 `initWGhostscript()`（每次 1.6-3.8 秒）。

---

## 五、压缩参数经验值（可直接抄）

```js
// 照片型三档(线上实测校准,"/60=体积优先,/46=均衡")
var PHOTO_TIERS = {
  low:  { dpi: 100, quality: 0.48, ptPxScale: 0.80 },  // 极端限制上传:约省 56%
  mid:  { dpi: 150, quality: 0.55, ptPxScale: 0.90 },  // 默认推荐:约省 46%
  high: { dpi: 200, quality: 0.72, ptPxScale: 1.00 },  // 打印存档:约省 18%
};

// 渲染倍率:绝不放大(放大只增体积不增信息)
function calcScale(vp1, tier) {
  if (vp1.width > 2200 || vp1.height > 2200) return tier.ptPxScale; // 像素当 pt 用的扫描件
  return tier.dpi / 72;
}
```

| 参数 | 经验值 | 说明 |
|---|---|---|
| 路由阈值 | 200 字/页 | 低于判照片型；课件通常 1000+ 字/页，很安全 |
| 渲染 dpi | 100 / 150 / 200 | 低于 100 文字发虚；200 适合打印 |
| JPEG 质量 | 0.48 / 0.55 / 0.72 | 0.5 是可读性下限，再低文字边缘糊 |
| 缩放 | 0.80 / 0.90 / 1.00 | 0.8 缩放对屏幕阅读无感，对体积很有效 |
| 无收益阈值 | 输出 < 原大小 × 0.98 才采用 | 否则保留原文件，标注"已优化" |

---

## 六、症状 → 决策速查表

| 症状 | 第一个该查的 |
|---|---|
| 三档输出完全一样大 | 参数键名是否一致（坑 7）；日志打出实际生效值 |
| 压缩率 0-1% | 文件结构（MediaBox / 有效 dpi / 滤镜类型），再怀疑引擎（坑 2） |
| `Worker 异常：未知` 且 message 空 | worker 顶层静态 import 失败 → 改动态 import（坑 4） |
| 页面正常但子资源 403 | 子资源 URL 缺 sharecode / token（坑 9） |
| 部署成功但行为是旧的 | CDN 缓存 → 查 `EO-Cache-Status`，加 `?v=`（坑 8） |
| 文字 PDF 变图片 | `getPage()` 漏 await（坑 6） |
| 元素"存在但看不见" | `display` 覆盖了 `hidden`（坑 10） |
| 自动化 upload 后页面变 about:blank | 用 CDP `DOM.setFileInputFiles`（坑 11/12） |
| 演示/测试数据看不出效果 | 测试素材太"干净"（坑 13） |
| `409 Git Repository is empty` | 空仓库先做一次初始提交（坑 14） |
| 空仓库 PUT contents 404 | 别传 `branch` 参数（坑 15） |

---

## 七、做对的事（值得复用的五个做法）

1. **页内日志面板优先于 console**：日志渲染进 DOM → 自动化可读、用户可截图、可复制粘贴给你。这是"线上闭环"的地基。
2. **哨兵消息**：worker 加载完成立刻 `postMessage`，让"模块解析成功"与"业务逻辑失败"可区分。
3. **依赖按需加载**：17MB 的 Ghostscript 只有文字型才拉，照片型首屏 1.7MB 就够。
4. **双管线按内容类型分流**：而不是一套参数硬套所有文件（这是本项目的核心价值）。
5. **改代码必改版本号**：把 `ASSET_VER` 和 `?v=` 写进 README 的部署章节，变成流程而不是记忆。

---

## 八、做得不好的地方（下次改进）

1. **本地环境耗了太久**：40 分钟在修 server 和浏览器工具。下次拿到任务先问"能不能直接上线测"，能就直接上。
2. **校验器没先验证**：`isValidPdf` 的 `Uint8Array.toString` bug 让我误判"3 档全失败"，多走一轮。**写校验器先跑一个已知好样本**。
3. **示例素材一开始没按真实数据做**：改了两次才让压缩率好看。做演示数据要先问"真实文件长什么样"（分辨率、噪点、光照）。
4. **`try/catch` 包探测代码时没把 catch 当日志信号**：差点让"文字 PDF 被渲染成图片"上线。
5. **参数 fallback 掩盖了键名错误**：`|| PHOTO_TIERS.mid` 让三档静默退化成同档。**fallback 必须配告警日志**。

---

## 九、交付清单（这类小工具站一份就够）

```
index.html / css/style.css
js/app.js               文件收集 / UI / 调度 / ZIP / 示例生成
js/compress.worker.js   类型探测 + 双管线
vendor/{pdfjs,pdf-lib,jszip,wghostscript}/
screenshots/            首页 + 结果页(README 引用)
README.md               网址 + 截图 + 架构 + 实测数据 + 踩坑表 + 调试方法
```

线上链接、GitHub 仓库、截图三样齐了才算交付完；README 里的实测数据要写清"哪类文件、哪个档位、原始多大、压完多大"。
