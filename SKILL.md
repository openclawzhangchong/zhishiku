---
name: zhishiku
description: 查询个人经验知识库（33 篇真实踩坑实录）。在打包/发版/真机调试/git 推送/AI 协作等场景开工前查一次；遇到报错先用错误串查 errorStrings.json。
---

# zhishiku 经验知识库查询

本 skill 接入个人经验知识库（`openclawzhangchong/zhishiku`，33 篇真实踩坑实录）。

## 知识库位置

本地工作副本：`C:\Users\Administrator\zhishiku-wt\`

**先更新再查**（内容可能被其他会话更新过）：
```bash
cd C:/Users/Administrator/zhishiku-wt && git pull
```
（GitHub 被墙时走 ECS 隧道：见该仓库 `29_GitHub访问` 文档，或项目内 `scripts/gh-tunnel.py`）

## 检索方法（按优先级）

1. **错误串匹配**：拿到报错后，用报错原文查 `errorStrings.json`（如 `EBUSY`、`WinError 5`、`Connection was reset`、`UAC.dll_unloaded`）→ 命中的 docs 编号
2. **症状/关键词匹配**：用症状描述查 `index.json` 的 `symptoms` / `keywords` 字段（中英文都试）
3. **场景必查**：下列场景**开工前**先读 `whenToRead` 匹配的文档：

| 场景 | 必读 |
|---|---|
| 桌面打包 / 覆盖安装前 | 30 |
| Android 真机安装 / 真机调试 | 31 |
| Android Web Audio / 音效功能 | 32、33 |
| git push 被墙 | 29、3 |
| 与 AI 协作方式拿不准 | 01 |
| 发版前后 | 02 |

## 使用规则（给 AI 的指令）

1. 命中文档后：先用 grep 找该文档的**「可复用教训」**行——这是一句话结论，直接可用
2. 需要细节（证据链/根因/具体命令）再读全文，不要一次全读（节省上下文）
3. 文档里的做法都标注了验证状态（真机实测/已撤回），**优先采信已验证的**
4. **解决新问题后必须回写**：按 `_TEMPLATE.md` 模板新增条目或补充到现有条目，commit + push——这是知识库持续增长的机制
5. zhishiku 是**公开仓库**，写入前必须脱敏（token/密码/内网 IP/登录态一律不入库）

## 文件结构

- `index.json` — 机器可读索引（33 篇元数据：stack/keywords/symptoms/whenToRead/oneLiner）
- `errorStrings.json` — 报错串 → 文档编号映射
- `00_总索引 · 症状速查导航.md` — 人读导航（症状速查表 + 分批来源）
- `_TEMPLATE.md` — 新条目模板
- `NN_主题 · 关键词.md` — 33 篇经验文档（每篇按「症状→证据链→根因→修复→可复用教训」结构化）
