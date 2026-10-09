# 06 · Web 前端与单文件 HTML 工程

> **摘要**：网页版、课件、组卷系统、小游戏都是**单文件 HTML** 形态（零构建、可 `file://` 双击打开、可微信发给同事）。单文件带来便携，也带来"改一处吞掉半个页面""用户看到的是缓存旧版"这类专属坑。本篇记录单文件工程的取舍、零侵入注入方法、DOM/可见性排查与缓存防呆。
>
> **关键词**：单文件 HTML、零外链、零侵入注入、MutationObserver、DOM 嵌套、祖先链 display:none、TDZ、缓存防呆、版本水印、IndexedDB
>
> **来源**：xiaole-web2027、xinxibook、jiaoxue、youxi、2026-09-29 工作台

---

## 一、单文件 HTML 的取舍（明确评估过）

**结论：保持单文件，不拆分**（app.html 约 141KB → 226KB / 5634 行）。

| 理由 | 说明 |
|---|---|
| 零构建 | 不需要任何打包工具 |
| 可 `file://` 双击调试 | 拆分后必须起服务器，调试门槛翻倍 |
| 便于分发 | 一个文件发给同事/学生即可 |

**代价的替代方案**：
- 写 `docs/APP_HTML_MAP.md` **行号地图**，用行号定位代替 IDE 搜索。
- **拆分阈值**（自主评估）：总行数 7500 / 有效 JS 4500；故障排查优先。
- 单文件课件体积建议 **107KB–528KB，>3MB 必须走图片优化**（见 08）。

---

## 二、零侵入注入（给已有页面加功能）

- 只在 `</body>` 前**追加一个独立 `<script>`**，原有代码**一行不动**。
- **MARK 幂等开关**：模块注释里的 MARK 字符串必须与脚本里的 MARK **完全一致**，否则会重复注入（出现过第一章被注入两份模块）。改注释必改 MARK。
- **判定 hook**：MutationObserver 盯 `.feedback, .q-feedback`，三级判定（class `ok/no` → innerHTML `fb-ok/fb-no` → 兜底取纯文本**前 30 字**），500ms 内同元素同结果去重。
- **音效 Web Audio 实时合成**（不引音频文件）：对 = C5-E5-G5 上行大三度（triangle）；错 = Bb3-F3 下行（sawtooth + 2600Hz lowpass）；`pointerdown` 首次交互解锁 AudioContext。
- **校验三件套**：`difflib` 证明**单一纯插入** + `node --check` + 真实场景测试。

### 注入类专属坑
| 现象 | 根因 | 修法 |
|---|---|---|
| 按钮不可点击 | 目标容器是 `<div style="display:none">`，**全篇无代码显示它** | 先查容器是否被真正显示；按钮插成**前一个兄弟节点** + 初始隐藏 + MutationObserver 盯答题反馈后显形。放容器外还顺带解决 textContent 污染 |
| 朗读把按钮文字也念出来 | 按钮 append 进文本容器，克隆件带着按钮 | 克隆后移除**所有 `button`**（不只自己的类）。凡"读页面文本再处理"的功能，UI 插在文本容器内必被自己读到 |
| 每个按钮都读最后一道题 | 循环里 `var blk` 被所有闭包共享 | 把 `blk` 作为**参数**传给回调。闭包共享单测验不出来，**必须人工过一遍逻辑** |
| 备份恢复后全部回到原始状态 | `sorted()[-1]` 按中文字符串排（「朗」<「音」），最旧的排最后 | 从目录名抽 `(\d{8}_\d{6})$` 排序 + **校验基准内容**（必须含音效、不含朗读） |

---

## 三、可见性排查：元素"存在但看不见"

**血案**：网页 `/app` 上开发者密码门、运行日志、清空确认、来电层、全局 toast **全部不可见**。
- 根因：`app.html:1992` 起 `<div class="phone-screen">` 到 2238 行**少一个 `</div>`**；浏览器自动补齐后把这些组件吞成 `#settingsSheet`（随时 `display:none`）的子元素。
- 修复后：密码门 0×0 → 430×247，来电层 430×932。

**方法**：
1. **判"是否真可见"不能只看自身 display，必须走祖先链找第一个 `display:none` 的祖先**。
2. 定位 DOM 嵌套错：用 Python `html.parser` 跑开闭标签栈，比肉眼数 div 快得多。

**其他前端坑**
| 现象 | 根因 | 修法 |
|---|---|---|
| 启动报 ReferenceError | **TDZ**：初始化代码引用了后面才 `let` 声明的变量（2537 行） | 启动渲染必须放脚本末尾（所有 let 声明之后） |
| 加字段后重载丢失 | `xdbLoadAll` 的字段**投影**没同步加 | 存储层加字段：**写入 + 读出投影必须同改**（`xdbAddRaw`/`xdbPut`/`xdbLoadAll` 三处） |
| 弹层只显示遮罩不显示内容 | 原型 CSS 的显隐类是 `.active` 不是 `.open` | 复用原型 CSS 前 **grep 关键类名** |
| 底部 tab-bar 被裁 20px | 遗留 `body { padding: 20px 0 }`（只有 ≤420px 媒体查询才归零） | `getBoundingClientRect` 偏移但 computed margin=0 → **优先查 html/body 的 padding** |
| 导出文件无反应 | `blob: URL + a[download]`，多数 WebView 宿主不处理 | Electron `session.on('will-download')` 接住，**网页零改动**。宿主层能兜的别改前端 |
| Mixed Content 被拦 | `mqtt.connect(worker + '/mqtt')`（https）被降级成 `ws://` | 显式转 `wss://`；**给 `mqtt.connect` 打桩要整体替换 `window.mqtt` 对象** |
| 下载文件名乱码 | 未用 RFC 5987 | `Content-Disposition: attachment; filename*=UTF-8''<encodeURIComponent(name)>` |
| 隐藏元素后整页 JS 崩 | 隐藏后 `markDirty` 仍绑定该 id → `$("cfg-cli")` 为 null → `null.addEventListener` | **防御式绑定**（元素存在才绑）+ 失败回退默认值，永不白屏 |
| fetch 永久 pending 像卡死 | 无超时 | 15s `AbortController` + 按钮禁用态 |

---

## 四、缓存防呆（"用户看到的 bug 其实已修"）

- 静态数据一律 **cache-busting**：`data.js?v=YYYYMMDD`。
- 页面显示「题库版本 X · 共 N 题」，导出脚本自动刷新版本号。
- 答题卡加「卡版 vX.Y.Z」灰色水印；桌面壳加 `APP_VER` 徽标 + 控制台输出。
- 前端版本要**可自证**：版本号 + 控制台输出 + 防缓存 meta + `Cache-Control: no-store`。
- 空值回落默认并回写（localStorage 存了空口令也要能恢复）。
- 改前端 vs 改后端的区别要写进交接文档：静态页每次请求读盘 ⇒ **改前端不用重启**；改 `server.js` 必须重启服务。

---

## 五、部署清单核对

- **部署前必须核对"部署物清单 == 源码清单"**。真实事故：`vendor/mqtt.min.js` 线上有（HTTP 200）、git 源码没有 → 直接部署会把线上的挤掉。
- 部署目录固定内容（如 `app.html` + `index.html` + `vendor/libopus.js` + `vendor/mqtt.min.js`）。
- **测试脚本绝不上传**（`_verify.js` 等）。
- 移动端适配：PC 舞台 `1280×720` 等比 scale；手机竖屏 `@media (max-width: 768px)` 降栏数、卡片单列、按钮加大、`safe-area-inset-bottom`。
- **竖屏溢出用数值断言**，不靠眼睛：`eval` 检查 `wrap.scrollHeight - wrap.clientHeight`，逐视口跑（靠肉眼漏过 318px 溢出）。

---

## 六、失败模式

1. **手改产物 HTML** → 下次重新生成全被覆盖（改生成脚本再重新生成）。
2. **在原字符串上边遍历边改** → 26 处引用错位（还原原貌后**一次性倒序替换**）。
3. **只看自身 display 判可见性** → 漏掉祖先链上的 `display:none`。
4. **不看退出码/完整日志就认为部署成功** → Pages 25MiB 超限静默失败（exit 0）。
5. **把中间态当结果**：端口在 listen 但没 `serve_forever`（手机页空白）；页面显示"已启用 20/20"实际下发 0。
6. **只复制部分代码做验证** → 漏掉加载期崩溃，误诊为"浏览器缓存"（验证必须跑**完整脚本**）。
7. **读剪贴板就"直接执行"** → 残留旧内容导致第二次点击无反馈（只能做"预填"）。
8. **无超时、无条件绑定** → 永久 pending 与整页崩溃。
