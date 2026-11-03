# AGENTS.md — AI 协作指引

> 任何 AI agent（ZCode / Claude / Cursor / 其他）打开本仓库工作时，**先读本文件**。

## 本仓库是什么

个人经验知识库：33 篇真实踩坑实录，按「症状 → 证据链 → 根因 → 修复 → 可复用教训」结构化组织。
覆盖：Windows 打包、Android 真机调试、Git 推送、AI 协作、移动端音效、PDF 处理等 30+ 主题。

**语言**：中文为主。**仓库公开**：写入前必须脱敏（token/密码/内网 IP/登录态/口令一律不入库）。

## AI 使用规范（必须遵守）

### 1. 开工前查库

接到任务后，若涉及以下高危场景，**先读对应文档再动手**：

| 任务涉及 | 必读 |
|---|---|
| Windows 打包 / 覆盖安装 / electron-builder | 30 |
| Android 真机安装 / 真机调试 / adb | 31 |
| Android Web Audio / 音效 / AudioContext | 32、33 |
| git push 被墙 / GitHub 访问 | 29、3 |
| 提示词 / AI 时效性问题 | 17 |
| 桌面自动化 / UIA | 19、12 |

### 2. 遇到报错先查错误映射

用报错原文（英文错误码优先）查 `errorStrings.json`：
- `EBUSY` / `WinError 5` / `UAC.dll_unloaded` → 30
- `Connection was reset` / `Failed to connect` → 29
- `InvalidStateError`（Web Audio）→ 32

命中后读对应文档的「可复用教训」表格行，**避免重新踩坑**。

### 3. 检索方法

```bash
# 关键词/症状检索（先更新本地副本）
git pull
grep -l "关键词" *.md                    # 找包含该词的文档
python -c "import json; [print(d['id'], d['file'], d['oneLiner']) for d in json.load(open('index.json'))['docs']]"
```

- `index.json`：33 篇元数据（id/title/stack/keywords/symptoms/whenToRead/oneLiner）
- `errorStrings.json`：报错串 → 文档编号映射
- 机器可读索引优先于人工浏览

### 4. 收尾必入库（知识库持续增长机制）

解决了新问题 / 验证了新做法后：

1. 复制 `_TEMPLATE.md` 改名 `NN_主题 · 关键词.md`（编号顺延，查现有最大编号）
2. 按六步表格填写（症状→错误归因→证据链→根因→为什么没早发现→修复→可复用教训）
3. **脱敏检查**：token/密码/内网 IP/口令不入库
4. 更新 `index.json`（新增 docs 条目）+ `00_总索引`（加一行 + 篇数）
5. commit + push（推送被墙见 29 号文档）

### 5. 文档纪律

- 每条标注**验证状态**（真机实测通过 / 已撤回 / 部分验证）
- 事故类按「看着像 A 其实是 B」写——错误归因也要记录
- 主题重复时在旧文档加「后续见 NN」交叉引用，不重写
- 版本号/路径/命令必须可执行（不要写占位符）

## 文件结构

```
00_总索引 · 症状速查导航.md    导航页（症状速查表 + 分批来源）
index.json                     机器可读索引（AI 用）
errorStrings.json              报错串 → 文档映射（AI 用）
_TEMPLATE.md                   新条目模板
SKILL.md                       ZCode skill 定义（可复制到 .zcode/skills/zhishiku/）
01~33_*.md                     33 篇经验文档
```
