# 05 · Flutter：0.x 锁版本 · abiFilters · golden 两类

> **摘要**：本机没有完整 Android 构建能力（沙箱拦截 java/gradle），CI 是唯一真值；本地只能靠 analyze + test + golden。本篇记录依赖冲突、ABI 体积、codegen 连锁报错、golden 抖动定性、以及一批 Dart 语言与 Riverpod 状态坑。
>
> **关键词**：pubspec 0.x 锁版本、abiFilters、APK 体积、build_runner、freezed、Riverpod、golden flaky、代理变量、analyze 改 lock
>
> **来源**：xiaoleapp、xiaole-app-xcode-2027

---

## 一、依赖与构建

| # | 现象 | 根因 | 修复 / 规则 |
|---|---|---|---|
| 1 | `flutter pub get` 版本求解失败 | `retrofit_generator` 要 analyzer >=13.3<15，`hive_generator` 要 <7，区间无交集 | 对**未被实际使用**的 generator 直接删，比降版本快 |
| 2 | CI `:workmanager:compileReleaseKotlin` 3 处 Unresolved reference | `workmanager: ^0.5.2` —— **caret 对 0.x 只允许补丁级升级（>=0.5.2 <0.6.0）**，被钉死在 2023 旧版 | **`0.x` 包绝不只写 `^0.x.y`**，去 pub.dev 查最新并显式锁版本（最终锁 0.8.0） |
| 3 | 升级后 `IOWebSocketChannel`/`crypto`/`http` 未定义 | 只改版本号，漏声明运行时依赖（原先是传递依赖） | 升级后 grep 所有 `package:xxx` import，逐个对照 pubspec 显式声明 |
| 4 | Riverpod 2→3 编译错 | Riverpod 3 移除 `StateNotifier` | 换基类为 `AsyncNotifier`/`Notifier`，**保持方法签名与调用面不动** |
| 5 | freezed 2→4 冗余 | freezed 3 自带 JSON 序列化 | 移除 `part '*.g.dart'`，删陈旧产物，让 CI 侧 build_runner 生成（**codegen 产物绝不入库**） |
| 6 | `WeChatColors` 未定义 → 连锁出模型未实现、`_$XFromJson not found` | `main_page.dart` import 用了两级 `../../`（应为三级）→ build_runner 分析异常 | **codegen 类报错，真因常在别处的 import 路径/语法错 → 先看第一条 error** |
| 7 | Manifest merger 失败 | pubspec.lock 重解析把 audio_session 升到要求 minSdk 24 的新版 | minSdk 23→24（android/ 非红线可改）；**lock 变动要重查 minSdk 约束** |
| 8 | CI 磁盘满：`Free space left: 29 MB` + `libonnxruntime.so: No space left on device` | `sherpa_onnx` 为 4 个 ABI 各打包巨型 so | `ndk { abiFilters += listOf("arm64-v8a","armeabi-v7a") }`；**APK 85MB → 37MB**。规则：引入带原生库的插件**第一步就加 abiFilters** |
| 9 | CI `flutter: command not found`（exit 127） | "Free disk space" 步骤删了 `$AGENT_TOOLSDIRECTORY`（Flutter/Java 工具链所在） | 清理脚本**黑名单式收窄**，只清 dotnet/boost，禁止删工具链与 NDK |
| 10 | `checkReleaseAarMetadata` 失败：requires core library desugaring | 插件强制要求脱糖 | `isCoreLibraryDesugaringEnabled = true` + `coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.4")`。**新增插件要预料"脱糖 / AAR 元数据 / compileSdk"三类要求；一个失败可能有多个独立根因，修完一个必须重跑 CI** |
| 11 | 本地 `pub get` 报 `PathNotFoundException: .plugin_symlinks\battery_plus` | 沙箱禁 symlink（Windows 桌面平台插件链接） | **不影响 Android 构建与 analyze**（CI 在 Linux），`flutter analyze --no-pub` 规避。分清"环境噪音"与"真实错误" |
| 12 | 本地 7 次 gradle 构建全部失败 | 沙箱对 java/gradle 进程文件操作随机拦截 | **本机不做 gradle 构建**，CI 是唯一真相；本地只做 analyze / test / golden |

---

## 二、Dart 语言与状态管理坑

| 现象 | 根因 | 规则 |
|---|---|---|
| `Expected an identifier` + `Expected to find ':'` | 写了 `j['result']?['tools']` —— **Dart 没有 `?[]` 空安全下标运算符**（只有 `?.`） | Map 深取值一律 `(x is Map) ? x['k'] : null` 逐层判空 |
| `override_on_non_overriding_member` | 插入字段时把 `@override` 与 `initState` 割裂 | **插成员前看上一行** |
| `const_eval_method_invocation`（4 处） | 把 `const TextStyle(...)` 换成 `AppTextStyles.body.copyWith(...)`，**外层 `const` 没删** | 常量→方法调用转换后必删外层 const；**不能只信 agent 自查**，必须跑 analyze |
| 恢复云端数据后列表不刷新 | `applyPullResult()` **直接写 SharedPreferences，不碰 provider**；且 provider 非 autoDispose，重进页面也不重读 | **绕过 Riverpod 直接写存储 = 内存态不刷新**；任何写盘路径必须显式 `ref.read(x.notifier).loadX()` |
| 一键云端"打包完成后卡死" | `sublist(i*250,(i+1)*250)` **end 未收口** → RangeError；且 `_push/_pull/_bind` 无 try/catch → `_busy` 永久 true | **切块必写 `math.min((i+1)*n, len)`**；所有网络方法必须 try/catch 保 `finally` 复位 |
| 头像配置永不生效 | ① `_lazyLoad()` 同步读已进缓存，紧接着 `read()` 无条件用默认值**盖掉真值并钉死**；② `_lazyLoad()` 在 `build()` 中同步 `revision.value++` → `markNeedsBuild() called during build` | 占位默认值与权威值必须有明确优先级；**任何 ValueNotifier 通知不得在 build 期间同步发出**（推迟到微任务） |
| 电话日志首次记录覆盖全部历史 | `recordXxx` 是同步 API，落盘前忘了 `await load()` | 旁路记录层：**写前必 load、清必带 generation** |
| Edit old_string 图省事包住整个方法体 | 插入字段时把 `_tailGuardMs` 方法体误删 | **old_string 只包锚点本身，new_string 完整带回锚点周边**；改完立即 grep 验证相邻结构 |
| TTS 音量无法控制 | `flutter_pcm_sound` 无 setVolume API | 播放前对 PCM Int16 做软件增益 + clamp 限幅 |

---

## 三、golden 截图测试

### 3.1 两类失败必须分开定性
| 类型 | 特征 | 处置 |
|---|---|---|
| **合法演进** | 大幅 diff（如 22.31% / 1,174,574px），能定位到版本号变更导致重排 | 对照实验定性（把 appVersion sed 回旧值 → 全绿；改回 → 必失败）→ `--update-goldens` |
| **时序 flaky** | 数值每次略变（0.28% / 14,706px），位置固定在 `FutureBuilder` 链上 | **golden 无变化却报错 = flaky**；判据：`git status test/goldens/` 始终为空 |

- **版本号出现在 golden 页面上时，每次发版必然抖动** → 更新 golden 是发版固定步骤，不是回归。
- `--update-goldens` 后紧接复跑仍报同样 diff → **连续执行两次**（首次因编译缓存未刷新写入旧基线）。
- 失败会生成 `test/failures/*_masterImage.png`（基线）与 `*_testImage.png`（新渲染）→ **直接读图目视核对**，看 `isolatedDiff` 确认 diff 只落在目标区域（无布局位移）才更新。
- FutureBuilder 多跳异步链在固定 pump 下必漂移 → 展示层改**单跳只读**，矫正逻辑移到别处。

### 3.2 运行前置条件
```
export PATH="/c/Users/Administrator/.workbuddy/binaries/flutter/flutter/bin:$PATH"
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY -u all_proxy -u ALL_PROXY
export PUB_HOSTED_URL=https://pub.flutter-io.cn FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
必须非沙箱运行
```
> **跑任何 flutter 命令前一律 unset 全部代理变量（含大小写）**——环境注入的 http_proxy 会劫持 flutter_tester 本地 WebSocket，报 `Invalid WebSocket upgrade request`。

---

## 四、本机环境规则

- **验证分工**：`flutter analyze` + 全量 `flutter test` 是本地唯一质量闸门；**Android 构建只信 CI**。
- **本地无 Flutter 时的类型错误必然漏到 CI**（如 `DateTime?` 传入非空 `DateFormat.format()`）→ 接受 1~2 轮 CI 构建失败作为正常成本。
- `flutter analyze` 会**顺带改写 `pubspec.lock`**（镜像源 209 处改回 pub.dev、顺手升版本）→ **提交前 `git checkout pubspec.lock`**。
- 多个子 agent 并行跑 flutter 会抢 `.dart_tool`；沙箱间歇拦截 `wmic.exe`/`reg.exe` 导致 `flutter test` 起不来（analyze 不受影响）→ 失败重试往往即成功。
- 大陆环境：`PUB_HOSTED_URL` / `FLUTTER_STORAGE_BASE_URL` 走 flutter-io.cn 镜像；**Gradle 不认 http_proxy 环境变量，国内镜像（maven.aliyun）是唯一稳路**。

---

## 五、测试纪律

- **每修一个 bug 就地加一条测试**（test 数从 25 → 71 就是这样涨上来的）。
- **写"约束型测试"要直接断言验收标准本身**，不要用间接代理指标：
  - 反例：`白字 ⇒ 底色亮度 < 0.45` 会放过 2.1:1 的对比度
  - 正例：直算 `(L亮+0.05)/(L暗+0.05)` 两端 ≥ 4.5:1
- **负向对照证明断言有牙齿**：故意把值改成会被旧断言放过的值（如 `0x7FAEF5`），新断言必须立即报错。
- **回归测试要跑完整 happy path**（含可选字段），只测拒绝路径必漏。
