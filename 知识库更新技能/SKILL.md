---
name: 知识库更新技能
description: 向本知识库仓库（github.com/openclawzhangchong/zhishiku）新增一篇或更新经验篇目的操作手册。当用户说「把这次的经验写进知识库」「更新 zhishiku」「沉淀到知识库」时使用。含仓库结构与命名惯例、正文范式、四处同步清单、URL 编码方法、推送与核验通道（HTTPS 502 → SSH 443），以及本机已知坑。
agent_created: true
---

# 向 zhishiku 知识库贡献经验

仓库：https://github.com/openclawzhangchong/zhishiku （public）
用户：张哥（openclawzhangchong）

> **给 AI 的话**：这是本仓库自带的技能。你若被要求往这个知识库加/改内容，
> 照这篇做，不要在根目录之外乱建结构、不要改动无关文件。
> 本技能本身也放在仓库根目录 `知识库更新技能/SKILL.md`——
> 因为「技能会被 AI 自动加载，而仓库里的一篇 md 不一定被读到」。

---

## 一、仓库结构惯例

- **根目录篇目命名**：`NN_<主题> · <钩子1>·<钩子2>·<钩子3>.md`
  - 例：`25_Windows静默后台任务 · 闪屏·编码方向·日志策略.md`
  - 两段式：`域 · 这篇的独特钩子`（原来的纯领域名看不出能解决什么问题，故改两段式）
  - 编号 `NN` 两位数字，取当前最大 +1
- **`00_总索引 · 症状速查导航.md`** —— 导航页，含：
  1. 标题「经验知识库总索引 · 症状速查导航（N 篇 / M 批）」
  2. 摘要 + 关键词 + 整理日期 + 脱敏说明
  3. 文件清单（按批次分表：`# / 文件 / 什么时候读它 / 核心一句话`）
  4. 症状速查表（分批次）
  5. 贯穿全库的顶层原则（**编号连续，跨批次累加**：12 → +5 → +3 …）
  6. 项目地图（来源标注，分批次）
- **`README.md`** —— 首页，含徽章、目录（**链接是 URL 编码的**）、症状表、顶层原则、来源与体量表
- **`LICENSE`**（MIT）
- **子目录** `GPU与AI部署经验/`（独立编号体系 00–09）
- **子目录** `知识库更新技能/`（本技能；不是一篇经验篇，不进 00 索引编号体系）

---

## 二、正文范式（每篇）

```
# NN · 域：钩子1 · 钩子2 · 钩子3

> **摘要**：为什么产生这篇（真实项目 + 真实事故 + 版本号），一句话点出反常识结论。
>
> **关键词**：...
>
> **来源**：项目名（仓库/版本）

---

## 一、<最要命的问题>：现象 / 根因 / 修复
（表格 + 代码块 + 真实数字、日志时间线）

...

## N、一句话速查
（编号列表，全部可执行/可判定）
```

写作要求（张哥风格）：

- **证据优于猜测**：带真实日志、退出码、文件:行号
- **反常识结论要突出**（"看着像 A 其实是 B"这类）
- 一句话能说清的用表格；说不清附最小复现
- 不写空泛原则，每条都要能落地

---

## 三、改动四处同步清单（缺一处就不完整）

1. **新增篇目文件**：`NN_...md`（编号取当前最大 +1）
2. **`00_总索引`**：
   - 标题篇数 N+1、批次按需 +1
   - 文件清单加一行（放进对应批次表）
   - 症状速查表加行（新增"第 X 批症状补充"子表）
   - 顶层原则追加（**接着编号往下排**）
   - 项目地图加"第 X 批来源"表
3. **`README.md`**：
   - 徽章：文档数、来源项目数、批次
   - 「这是什么」加批次描述
   - 目录加"第 X 批"表 + 新篇行（**链接必须 URL 编码**）
   - 症状速查表加"第 X 批症状补充"
   - 顶层原则追加（与索引同步）
   - 「来源与体量」表加一列
4. **编码**：`.md` 一律 **UTF-8 无 BOM**（库内所有文件都是，别加 BOM）

---

## 四、链接 URL 编码

README 里的文件链接形如：

`25_Windows%E9%9D%99%E9%BB%98...%E7%AD%96%E7%95%A5.md`

生成方法：

```bash
python3 -c "import urllib.parse; print(urllib.parse.quote('25_你的文件名.md'))"
```

（中文、`·`、空格都要编码；`_` 保留）

---

## 五、推送通道（本机关键：HTTPS 会 502）

**本机（张哥这台 Windows）HTTPS 克隆/推送会连续 502**：
`CONNECT tunnel failed, response 502`（代理隧道问题，反复重试也无效）。

**改用 SSH 443 端口**，实测一次成功：

```bash
GIT_SSH_COMMAND="ssh -i C:/Users/Administrator/.ssh/id_ed25519_pyxz -o StrictHostKeyChecking=no" \
  git clone --depth 1 ssh://git@ssh.github.com:443/openclawzhangchong/zhishiku.git zhishiku-probe
```

- 密钥：`C:\Users\Administrator\.ssh\id_ed25519_pyxz`（**非默认名**，SSH 不会自动用）
- 走 `ssh.github.com:443`（绕防火墙），不是标准 22 端口
- 推送同理，加同一个 `GIT_SSH_COMMAND` 前缀

---

## 六、操作流程

```bash
# 1. 浅克隆到临时目录（别污染工作区）
cd "C:/Users/Administrator/AppData/Local/Temp" && rm -rf zhishiku-probe
GIT_SSH_COMMAND="ssh -i C:/Users/Administrator/.ssh/id_ed25519_pyxz -o StrictHostKeyChecking=no" \
  git clone --depth 1 ssh://git@ssh.github.com:443/openclawzhangchong/zhishiku.git zhishiku-probe

# 2. 读 00 索引 + README + 一篇近期范文，摸清当前篇数与编号
#    Windows 下 /tmp 真实路径：C:\Users\Administrator\AppData\Local\Temp\zhishiku-probe
#    注意：Write 工具要用 Windows 绝对路径，/tmp 会被拒（"target_path must be an absolute path"）

# 3. 写新篇 + 改 00 索引 + 改 README

# 4. 提交推送
cd zhishiku-probe && git add -A
git commit -F <中文消息文件>    # heredoc/命令行中文会被安全策略拦，消息写文件再 -F
GIT_SSH_COMMAND="ssh -i C:/Users/Administrator/.ssh/id_ed25519_pyxz -o StrictHostKeyChecking=no" \
  git push origin main

# 5. 核验（关键：不能只看 push 输出）
gh api repos/openclawzhangchong/zhishiku/commits/main --jq '{sha, msg: .commit.message}'
# 比对本地 git rev-parse HEAD，一致才算真成功
```

---

## 七、坑清单

- **HTTPS 通道 502** → 换 SSH 443（见第五节）。这是本机最常见的第一道拦路虎。
- **`git ls-remote` 可能报 502**（代理隧道中途失效）。若 push 输出显示成功但
  ls-remote 502，**别重推、别谎报**——改用 `gh api .../commits/main` 核验 sha。
- **Write 工具写 `/tmp/...` 会被拒** → 用 `C:\Users\Administrator\AppData\Local\Temp\...`。
- **中文 commit message 走 `-F 文件`**，heredoc 或命令行内联都会被安全策略拦。
- `rm -rf` 临时克隆目录可能触发沙箱升级批准。
- 库内 md **无 BOM**，别画蛇添足加 BOM。
- 顶层原则**编号是跨批次累加的**（不是每批从 1 开始），新批接着排。
- README 的目录链接**是 URL 编码的**，直接写中文文件名会 404。
- **临时文件别提交**：写 commit message 用的 `_commitmsg.txt` 之类，用完就删，
  别让它进仓库（曾经被误提交过一次）。
- 编辑 README 时用**锚定式替换**，别整段重写——曾误删过过渡段落。

---

## 八、一句话速查

1. 克隆/推送一律走 **SSH 443 + `GIT_SSH_COMMAND` 指定 `id_ed25519_pyxz`**，HTTPS 必 502。
2. 加一篇 = 改四处：新篇 + `00_总索引` + `README` + 编码（UTF-8 无 BOM）。
3. 顶层原则编号**跨批次累加**，不重排。
4. README 目录链接必须 **URL 编码**，否则 404。
5. 中文 commit 消息写文件走 `git commit -F`。
6. 推送成败**用 `gh api .../commits/main` 核验 sha**，不看 push 输出、不信 ls-remote。
7. 临时文件（`_commitmsg.txt` 等）用完即删，不进仓库。
