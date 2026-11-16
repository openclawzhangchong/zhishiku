# 43 · Tauri NSIS 安装器美化：图标换成真图标 · 全中文 · 自定义欢迎页

> **摘要**：Tauri 打出来的 Windows 安装器默认长相很"原始"——exe 图标是 NSIS 蓝色地球、界面全英文、欢迎页标题是长产品名折成两行。这些**都能配**，不用改 Tauri 源码：`installerIcon/sidebarImage/headerImage` 换素材，`languages` + 自定义语言包换中文，`template` 覆盖欢迎页文案。三个坑：语言包 **BOM**、NSIS 换行必须写 `$\r$\n` 字面量、`template` 需复制一份生成好的 nsi 再改。

**关键词**：tauri, nsis, installer, installerIcon, sidebarImage, headerImage, MUI, SimpChinese, customLanguageFiles, template, nsi, MUI_WELCOMEPAGE_TITLE, BOM, $\r$\n, 中文安装包, 安装器美化

---

## 主题 1：exe 图标是 NSIS 地球 → 换成产品图标

| 项 | 内容 |
|---|---|
| **症状** | 安装包 exe 在资源管理器里显示 NSIS 默认的蓝色地球图标，不像自家产品 |
| **根因** | `bundle.windows.nsis.installerIcon` 为空（Tauri 模板里 `!define INSTALLERICON ""`） |
| **修复** | 用 PIL 从 1024 源图生成多尺寸 ico（16/32/48/64/128/256），配 `"installerIcon": "icons/nsis/installer.ico"`（`uninstallerIcon` 同值） |
| **可复用教训** | ico 必须**多尺寸**，只放 256 的话任务栏/小图标视图会糊；PIL 的 `save(format='ICO', sizes=[...])` 一次搞定 |

**验证状态**：真机实测（`ExtractAssociatedIcon` 提取确认是产品图标）

## 主题 2：界面全英文 → 简体中文

| 项 | 内容 |
|---|---|
| **症状** | 欢迎页 "Welcome to ..."、按钮 Next/Cancel、维护页全英文 |
| **错误归因** | 以为 Tauri 只支持英文 |
| **证据链** | 生成的 `target/release/nsis/x64/installer.nsi` 里写死 `!insertmacro MUI_LANGUAGE "English"` |
| **根因** | `bundle.windows.nsis.languages` 未配置时默认 `["English"]` |
| **修复** | ① `"languages": ["SimpChinese"]`；② 写 `src-tauri/nsis-lang/SimpChinese.nsh` 覆盖 Tauri 自己的串（维护/卸载/错误提示），配 `"customLanguageFiles": {"SimpChinese": "./nsis-lang/SimpChinese.nsh"}`。生成后 `English.nsh` 消失、出现 `SimpChinese.nsh` 即为生效 |
| **可复用教训** | 🔴 **语言包文件必须无 BOM**——带 BOM 会让 makensis 在第 1 行报 `!include: error in script`，错误信息完全不提 BOM，极难查。Python 写文件用 `encoding='utf-8'`（不是 `utf-8-sig`） |

**验证状态**：真机实测（界面全中文）

## 主题 3：界面丑 → 侧边图 + 头部图 + 短标题

| 项 | 内容 |
|---|---|
| **症状** | 左侧是 MUI 默认的白色空白/通用图；欢迎页标题长产品名折两行 |
| **修复** | ① `sidebarImage` = 164×314 BMP（深色渐变 + 产品图标 + 产品名 + 中文副标题）；② `headerImage` / `uninstallerHeaderImage` = 150×57 BMP（小图标 + 产品名）；③ 用 `template` 覆盖欢迎页文案 |
| **尺寸硬约束** | 侧边图必须 **164×314**、头部图 **150×57**，且都要 **BMP**（PNG 不行） |
| **可复用教训** | 标题折行是因为 MUI 欢迎页文案是 `欢迎使用 $(^NameDA) 安装程序`，`$(^NameDA)` 就是完整产品名。**不要改产品名**（会影响安装目录/快捷方式，老用户会装成两份），改用 `template` 覆盖标题 |

**验证状态**：真机实测（截图确认三个元素都渲染）

## 主题 4：自定义 template 的两个必踩坑

### 坑 A：`$\r$\n` 被吃掉

| 项 | 内容 |
|---|---|
| **症状** | 自定义 `MUI_WELCOMEPAGE_TEXT` 后 makensis 报 `unterminated string parsing line at installer.nsi:168` |
| **根因** | 通过 shell heredoc / PowerShell 拼字符串时，NSIS 的 `$\r$\n` 转义序列被上层解释掉，落盘成了**真实换行**，字符串就断行了 |
| **修复** | 用 Python 的 **raw 字符串** `r'$\r$\n'` 拼接，避免任何层解释转义 |
| **可复用教训** | 写 NSIS 脚本里含 `$` 的内容，一律走「落盘脚本 + raw 字符串」，别在 shell 里拼 |

### 坑 B：template 从哪来

| 项 | 内容 |
|---|---|
| **做法** | 先正常打包一次（或让它失败），拿到 `target/release/nsis/x64/installer.nsi`，复制到 `src-tauri/nsis-template/installer.nsi`，只改需要的那几行，再配 `"template": "./nsis-template/installer.nsi"` |
| **注意** | 生成的 nsi 里那些本机绝对路径（`!define MAINBINARYSRCPATH "E:\..."`）**不用管**，Tauri 每次打包会重新填充；但模板一旦和 Tauri 版本绑定，升级 Tauri 后要重新同步一次模板 |
| **可复用教训** | template 是**最容易随 Tauri 升级失配**的定制点——改动越小越好（本例只加了 2 行 define） |

**验证状态**：真机实测

---

## 附：完整配置片段（tauri.conf.json）

```json
"windows": {
  "nsis": {
    "installerHooks": "./installer-hooks.nsh",
    "languages": ["SimpChinese"],
    "customLanguageFiles": { "SimpChinese": "./nsis-lang/SimpChinese.nsh" },
    "displayLanguageSelector": false,
    "compression": "lzma",
    "installerIcon": "icons/nsis/installer.ico",
    "uninstallerIcon": "icons/nsis/installer.ico",
    "sidebarImage": "icons/nsis/sidebar.bmp",
    "headerImage": "icons/nsis/header.bmp",
    "uninstallerHeaderImage": "icons/nsis/header.bmp",
    "installMode": "currentUser",
    "startMenuFolder": "WorkBuddy Switch2api",
    "template": "./nsis-template/installer.nsi"
  }
}
```



### 补充（2026-11-16 晚）：中文化漏网的终极排查法 —— 做差集

只改了 `languages=["SimpChinese"]` + 写 20 条 LangString 后，逐页截图发现**安装进度页
（WebView2 下载/安装提示）和卸载确认页（删除应用数据复选框）仍是英文**。
原因是 Tauri 模板实际引用 **24** 个 `$(xxx)` 字符串，我之前只覆盖了 20 个。

别靠肉眼一页页猜，直接做集合差集（一次找全）：

```python
import re
nsi  = open('target/release/nsis/x64/installer.nsi',encoding='utf-8',errors='replace').read()
lang = open('target/release/nsis/x64/SimpChinese.nsh',encoding='utf-8',errors='replace').read()
refs = set(re.findall(r'\$\(([A-Za-z0-9_]+)', nsi))          # 模板引用
defs = set(re.findall(r'LangString\s+([A-Za-z0-9_]+)', lang))    # 语言包定义
print('缺失:', sorted(refs - defs))   # ← 这些会显示英文，逐个补中文
```

补齐后重新打包，**必须重建语言包后再做一次差集**（用 `target/release/nsis/x64/` 下
生成的副本，不是源文件），确认缺失为 0 才发版。

**可复用教训**：任何"国际化没做全"的问题，先做「引用集合 − 已翻译集合」的差集，
比逐页截图找快且不会漏；截图只用于最后验收 UI 观感。

> 关联：`41_单文件安装包打包 · bat编码裂行·转义吃字符·NSIS树杀`（同属 NSIS 打包家族，那篇讲的是自建 NSIS + IExpress）。
