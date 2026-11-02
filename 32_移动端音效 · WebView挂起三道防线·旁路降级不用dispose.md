# 32 · 移动端音效：WebView挂起三道防线、旁路降级不用dispose、双版本并存

> **摘要**：把桌面端 8 款音效 + 10 段均衡器移植到 Android APK，卡点不是实现，是 **Android WebView 会在息屏/切后台/来电时挂起 AudioContext**，且 `createMediaElementSource()` 对同一 audio 元素**只能调用一次**——接管后永远回不到直连。方案：**默认关闭 + 显式开关 + 三道自愈防线 + 旁路降级**（增益归零=等效直连，绝不用 dispose）。配**双版本并存**发布策略（音效版与无音效稳定版同签名互覆盖），真机验收息屏 20s / 切后台 30s 全过。

**关键词**：Android WebView、AudioContext 挂起、createMediaElementSource 一次性、旁路降级、双版本并存、息屏自愈、真机验收

---

## 事故 1：简单放开 isElectron 门控 = 零自愈能力的音效链路

| 项 | 内容 |
|---|---|
| **背景** | 桌面端音效系统在 `setupEQ()` 内部调用 `setupContextStateMonitoring()`（AudioContext suspended 时自动 resume） |
| **错误归因** | "把 `if (!isElectron) return` 改掉就行" |
| **证据链** | ① `setupContextStateMonitoring()` 在 `setupEQ()` **内部** ② `setupEQ()` 在非 Electron 下第一行就 return ③ 所以移动端**从未建立过任何自愈机制** ④ 移动端已知 5 个挂起场景：息屏/切后台/来电/蓝牙切换/长时播放被 ROM 回收 |
| **根因** | 门控不是"能不能建图"的问题，是"建图后**挂在系统层**"的问题。WebView 的 AudioContext 挂起是系统行为，代码不主动恢复就永远无声 |
| **修复** | 三道防线（见下）+ 默认关闭 |
| **可复用教训** | **平台差异的门控放开 ≠ 功能移植完成**。桌面端隐式依赖的"环境永不挂起"在移动端全部要显式处理 |

## 事故 2：降级用 dispose() = 永久无声

| 项 | 内容 |
|---|---|
| **错误归因** | "出问题就 dispose() 重建" |
| **证据链** | `createMediaElementSource()` 文档明确：**同一 HTMLMediaElement 只能调用一次**；调用后元素输出永久绑定到该 AudioContext |
| **根因** | dispose 后想重建 → 二次调用必抛 InvalidStateError；不重建 → 元素输出已被绑到已 dispose 的 context，**无声** |
| **修复** | **旁路降级**：所有增益 `setTargetAtTime(0)`（tone/EQ/filters）、wet=0、pan 归零——信号仍流经 Web Audio 但等同直连。**保证永远有声**，只是失去音效 |
| **可复用教训** | Web Audio 接管媒体元素是**单向门**。任何"回退方案"必须设计成"参数归零"，而不是"拆除重建" |

## 事故 3：音效按钮藏在设置开关后面——真机验收第一反馈"UI 上看不到音效"

| 项 | 内容 |
|---|---|
| **症状** | APK 装好了，用户：**"我没在 UI 上看到音效"** |
| **错误归因** | "按钮没渲染"（CDP 验证：按钮在，被 `v-if="开关开启"` 挡了） |
| **根因** | 设计错误：把「先去设置开开关 → 重启 → 才见按钮」当成了合理流程。桌面端是**常显按钮**，移动端却多加了一道门槛 |
| **修复** | 按钮常显；**首次点击自动开启开关 + toast 提示重启生效**，与桌面体验对齐 |
| **可复用教训** | **入口常显，开关内联**。用户在播放页想调音效，不应该被要求先去设置。多一道"前置开关"就多一次"找不到功能"的反馈 |

---

## 实现要点（三道防线 + 门控）

```ts
// 门控：三处 isElectron 判断改为
const canUseWebAudio = isElectron || (isMobileShell() && isMobileSoundEffectEnabled());

// 防线1：回前台自愈（最主要时机）
document.addEventListener('visibilitychange', () => {
  if (!document.hidden) ctx.resume().catch(() => fallbackToBypass());
});
// 防线2：播放中 30s 健康检查（兜底 ROM 回收）
// 防线3：连续 3 次 resume 失败 → fallbackToBypass()
//   （tone/EQ 增益归零、wet=0、pan 归零 —— 全部 setTargetAtTime 平滑，不断链）
// 注意 dispose 里清理健康检查定时器；statechange 里 running 时计数归零
```

**开关改动需重启 App**：链路无法运行期重建，UI 上必须明示（toast + 验收清单写明）。

---

## 双版本并存策略（音效有风险时的发布形态）

| 版本 | 定位 | 传播渠道 |
|---|---|---|
| `2.0.0`（音效版） | 推荐版，功能全 | GitHub Release（正式，非 prerelease）+ 下载页置顶 |
| `1.6.0`（无音效稳定版） | 经典稳定版，长期保留 | 下载页第二位 |

- **同签名可互覆盖安装**，数据（登录态/收藏）互通保留 → 用户可随时切换
- 下载页每项带标签说明（如「⭐ 推荐版 · 内置音效」「🛡️ 经典稳定版 · 无音效」），家人/小白不会选错
- beta 阶段：稳定渠道（下载页）**绝不挂 beta**，beta 只走 GitHub prerelease + 桌面副本
- 真机验收通过 → beta 转正式（PATCH prerelease=false），下载页才更新

---

## 真机验收结果（realme / Android 16，2026-11-02）

| 测试 | 结果 |
|---|---|
| 安装 / 启动 | ✅ |
| 音效按钮显示（全屏页均衡器图标） | ✅ |
| 音效面板 9 按钮渲染 + 中文标签 | ✅ |
| 音效切换持久化（bassBoost / surround3d） | ✅ |
| **息屏 20 秒 → 亮屏** | ✅ isPlay 保持 true |
| **切后台 30 秒 → 回前台** | ✅ 进度 02:38→03:29 未中断 |
| 关闭开关重启回退 | ✅ |

> 关联：`31_Android真机调试 · scrcpy绕守护·CDP读store`（验收工具链）、`20_安卓外壳 · 三份副本`（APK 基础）、`22_流程质量 · HANDOFF十节`（交接文档结构）。
