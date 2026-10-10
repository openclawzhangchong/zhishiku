# 02 · 发版交付：tag 锚点与文档回填八步

> **摘要**：一人 + AI 高速迭代时，"能退回去"比"能跑起来"更重要。本篇记录版本 tag 体系、每版固定动作、CI 红灯的正确处置、文档回填纪律，以及红线分区制度——这些是把几十个版本迭代的通信层事故压到零的关键。
>
> **关键词**：git tag、锚点、回滚通道、CHANGELOG、README 版本档案、CI 红灯、交付序列、红线分区、质量门
>
> **来源**：xiaoleapp、xiaole-app-xcode-2027、xiaole-web2027、jiaoxue、xinxibook

---

## 一、tag 体系：回滚通道是发版的保险箱

| Tag 类型 | 命名 | 作用 | 触发 CI？ |
|---|---|---|---|
| 正式版本 | `v6.11.5` | 发布，构建 APK | ✅ 触发 |
| **开工锚点** | `anchor-pre-v6.11.5` | 指向上一个可用版本，**不以 v 开头** | ❌ 不触发 |
| 稳定锚点 | `stable-1.1.7` | 真机验证通过的回退点 | ❌ |
| 功能封存 | `archive-voice-wip-20260912` | 代码不删，随时找回 | ❌ |
| 修复标记 | `v1.1.5b` / `v1.9.6-portable-fix` | 失败版本的后续修复 | ✅ |

**规矩**
- 每次发布打**两个 tag**：版本 tag + 锚点 tag，形成双通道回退。
- **失败过的 tag 绝不复用**（v1.1.5 曾指向无产物的提交 → 新建 v1.1.5b）。
- 回退通道完整性要**定期 `git ls-remote --tags` 核对**。
- 版本号两处必须一致：`pubspec.yaml` 的 `version:` 与 `app_config.dart` 的 `appVersion`。
- 编号规矩：**发布前取"当时远端最高号 +1"**，防止与其他 AI 撞号或倒挂。

---

## 二、每版固定动作 8 步（第 8 步最常被省，也最容易出事）

1. 打锚点 tag `anchor-pre-vX.Y.Z`
2. 子 agent 实施（或自己实施）
3. **主控复核：必须跑 `flutter analyze`，不能只信 agent 汇报**
4. 本地 commit
5. 推送（API 四步法，校验远端 tree == 我的 BASE tree）
6. 打 `v*` tag 触发 CI
7. **确认 CI 绿**：失败必须拉日志定位，**绝不把失败 tag 当成功，也别一失败就改版本号重发**
8. ⭐ **文档同步与标记（不可省）**：README 状态列 / CHANGELOG / 进度文件 §存档表 / PLAN 头部横幅全部回填，**零「待验证」残留**

> 血案：v6.8.1 实际构建失败却被标成 ✅ CI 通过，而唯一可用回退点 v6.7.1 标成"待发布" —— **错误绿灯会误导回退决策**。

---

## 三、CI 红灯的正确处置

| 步骤 | 做法 |
|---|---|
| ① 先定位失败在哪一步 | 日志里若已打出 `✓ Built build/app/outputs/flutter-apk/app-release.apk (41.0MB)` 而失败在 `Create Release`（`uploads.github.com socket hang up`）⇒ **上游上传抖动，重跑即可，不改代码、不改版本号** |
| ② 区分失败语义 | `exit 143 / SIGTERM` = runner 被停机（不是代码问题）；`RemoteDisconnected` = 网络瞬断（不是 CI 失败），轮询脚本要加 retry |
| ③ 不阻塞交付 | CI 失败时先用本机 `flutter build apk --release` 产物喂饱 Release，云端成功后再替换官方包 |
| ④ 重跑 | `POST /actions/runs/{id}/rerun-failed-jobs` |
| ⑤ 回填 | 拿到结论立即回填 README：**✅ CI 通过（run xxx）** 或 **❌ 构建失败、无 APK、勿用** |

**tag 不触发 workflow 的三个原因**（连续踩了 3 次）：
1. **root commit（无 parent）的 tag 不触发**；
2. tag 指向的 commit 必须已在分支上；
3. workflow 文件必须已被 GitHub 注册（worktrees API 列表为 0 = 未注册）。
兜底：`workflow_dispatch`。

---

## 四、交付序列（发布链，一步不能少）

```
CI 验证 → APK 下载到桌面 → PLAN md 双仓回填 → git push 远程 → 真机验证
```
- APK 下载走**两段式**：带 `Authorization` 拿 302 Location，**第二段不带任何鉴权头**（带鉴权会干扰 Azure blob SAS 鉴权，返回 401）。
- 交付物命名：桌面 `xiaole-client-v<版>.apk`；**本机包禁止占用 `app-release.apk` 名**（会与 Release 资产混淆，改 `xxx-local.apk`）。
- 交付必附：字节数 + sha1、完整 URL、代码改动位置、表格化总结。

---

## 五、红线分区制度（把"能不能碰"制度化）

| 级别 | 范围 | 规则 |
|---|---|---|
| 🔒 **绝对冻结** | 身份生成工具、OTA 的 appName、音频编解码/采样率/缓冲参数 | 一个字都不能改 |
| ⚠️ **条件修改** | `lib/core/network/`（通信层） | 每次必须①对照协议速查表逐字段核对 ②真机冒烟 ③diff 逐行评审登记 |
| ✅ **自由区** | `lib/presentation/`、新增文件、`lib/data/`（加字段必须带默认值） | 随便改 |

**配套质量门**：
- **红线 diff**：`git diff <anchor> -- lib/core/ lib/data/` 必须为空（或仅含已批准例外）—— 一行命令把"我没碰通信"变成可验证事实。
- 新功能优先**新增调用路径，绝不修改现有通信代码**。
- 安全入口用"**恰好 N 处**"的可 grep 断言固化（如 `DevCenterPage()` 只允许 2 处且必须在密码门体内）。

---

## 六、文档纪律

- **CHANGELOG 倒序**记录，每版一段，含事故复盘。
- **README 版本档案**：徽章 + 档案行 + 状态列；拿到 CI 结论立即回填，严禁长期挂"待验证"。
- **进度文件**是唯一锚点，含：当前状态（基线 / 本地 HEAD / 远端 HEAD / 最新 tag）+ 每版状态表 + 固定动作 + 断电续接六步。
- 已知坏版本在 README **标红**：`v6.8.0 / v6.8.1 ❌ 构建失败、无 APK、勿用（带 run 号）`。
- 纯文档提交**不打 tag 不触发 CI**。

---

## 七、失败模式

| # | 反模式 | 后果 | 正解 |
|---|---|---|---|
| 1 | 打完 tag 才发现有改动没进去（"顺手 polish"） | 只能删 tag 重建 → 多轮 CI + 两个同名 Release | 所有 polish 在第一次打 tag 前做完 |
| 2 | README 状态列不回填 | 错误绿灯误导回退 | CI 有结论立即回填 |
| 3 | 一失败就改版本号重发 | 版本号通胀、历史混乱 | 先定位失败步骤，抖动就重跑 |
| 4 | 复用曾失败的 tag | 指向无产物提交 | 新建 tag |
| 5 | 两个 AI 并行改同一仓库 | 基准 commit 落后 → 一推覆盖对方成果 | 单写者原则 + 推送前校验远端 tree == base tree |
| 6 | 手工维护文件清单 | tree 校验失败（漏 `pubspec.lock`、漏删除项） | 用 `git status` / `ls-files` 生成清单 |
| 7 | 本机包叫 `app-release.apk` | 与 Release 资产混淆 | 加 `-local` 后缀 |
| 8 | 只看 tag 建成功不看 CI 触发 | tag 建了但没出包 | 发版后必须确认 CI 已触发 |
| 9 | 没有锚点 tag 就大改 | 出问题退不回去 | 动手前先打锚点 |
| 10 | 把未跑的 golden 说成通过 | 声明失实 | 声明前必须实跑，**绝不把失败说成成功** |
