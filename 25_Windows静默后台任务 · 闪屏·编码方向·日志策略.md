# 25 · Windows 静默后台任务：闪屏 · 编码方向 · 日志策略

> **摘要**：做一个 Windows「移动热点保活」工具（任务计划程序定时唤起 PowerShell 脚本），发 v1.0.0 后**跑了一天才暴露问题**——**屏幕每隔 5 分钟闪一下黑框**，同时日志一天涨了 288 行。排查后确认闪屏不是脚本 bug，而是**计划任务直接调 `powershell.exe` 的机制必然结果**：控制台程序在进程启动的最初几毫秒会创建控制台窗口，`-WindowStyle Hidden` 挡不住。修复方案是加一层 **VBScript 静默启动器**（`WScript.Shell.Run(cmd, 0, True)`）。过程中还踩到一组**方向相反的编码规则**（`.ps1` 要 UTF-8 BOM，`.vbs` 要 GB2312），以及"临时脚本不带 BOM 导致误判文件不存在"的坑。
>
> **关键词**：任务计划程序、闪黑框、无窗口、VBScript、WScript.Shell.Run、WindowStyle Hidden、UTF-8 BOM、GB2312、WinRT、PowerShell 5.1、日志膨胀、静默启动器
>
> **来源**：项目 `win-mobile-hotspot-keeper`（GitHub 公开仓，2026-10-10 v1.0.0 → 2026-10-11 v1.1.0）

---

## 一、闪屏：`-WindowStyle Hidden` 挡不住，必须从"创建进程"就无窗口

### 现象

计划任务每 5 分钟触发一次，**屏幕上每隔 5 分钟闪一下黑框**。
任务配置里**明明已经**设了 `<Hidden>true</Hidden>`，命令行也加了 `-WindowStyle Hidden`。

日志实锤触发频率（每 5 分钟准点一条）：

```
[07:25:04] [OK] 完成：热点已开启。
[07:30:01] [OK] 完成：热点已开启。
[07:35:04] [OK] 完成：热点已开启。
```

### 根因

> **`powershell.exe` 是控制台子系统（console subsystem）程序。**
> Windows 在进程启动的**最初几毫秒必然创建一个控制台窗口**，然后才应用 `-WindowStyle Hidden` 把它隐藏。
> 用户看到的，正是这一瞬间的窗口。

**`-WindowStyle Hidden` 是"先建窗口、再藏起来"，不是"不建窗口"。**
这跟脚本写得好不好无关，**在 PowerShell 脚本内部无法解决**——必须在**创建进程**时就指定无窗口。

### 修复：加一层 VBScript 静默启动器

`WScript.Shell.Run(cmd, windowStyle, waitOnReturn)` 的**第 2 个参数 `0`** 表示窗口完全不显示。
这是"从进程创建起就没有窗口"，屏幕上不会有任何闪烁。

`run-hotspot.vbs`（核心就一行）：

```vbs
Option Explicit
Dim fso, shell, scriptDir, cmd
Set fso   = CreateObject("Scripting.FileSystemObject")
Set shell = CreateObject("WScript.Shell")

' 自动取本 VBS 所在目录，拼出同目录下的 ps1（不硬编码绝对路径）
scriptDir = fso.GetParentFolderName(WScript.ScriptFullName)

Dim ps1Path, logPath
ps1Path = fso.BuildPath(scriptDir, "hotspot-on.ps1")
logPath = fso.BuildPath(scriptDir, "hotspot.log")

If Not fso.FileExists(ps1Path) Then WScript.Quit 2   ' 缺失时静默退出，不弹窗

Dim sysRoot
sysRoot = shell.ExpandEnvironmentStrings("%SystemRoot%")

cmd = """" & sysRoot & "\System32\WindowsPowerShell\v1.0\powershell.exe"" " & _
      "-NoProfile -ExecutionPolicy Bypass -File """ & ps1Path & """ " & _
      "-Quiet -LogFile """ & logPath & """"

shell.Run cmd, 0, True     ' 0 = 窗口完全不显示；True = 等待子进程结束
WScript.Quit 0
```

计划任务的 Action 相应从直接调 powershell **改为调 wscript**：

```xml
<Exec>
  <Command>%SystemRoot%\System32\wscript.exe</Command>
  <Arguments>//B //Nologo "C:\...\run-hotspot.vbs"</Arguments>
</Exec>
```

> `//B` = batch 模式（不弹脚本错误对话框）；`//Nologo` = 不显示版权头。

### 一句话铁律

> **要让 Windows 后台任务真正静默，只能在「创建进程」时指定无窗口
> （`CREATE_NO_WINDOW` / `Run(cmd, 0)`），任何"进程内隐藏窗口"的手段都会有闪烁。**

---

## 二、编码方向：`.ps1` 要 UTF-8 **带 BOM**，`.vbs` 要 **GB2312**

同一套工具里，两类脚本的编码要求**方向相反**，搞反就出玄学故障。

| 文件类型 | 正确编码 | 搞反的后果 |
|---|---|---|
| `.ps1`（PowerShell） | **UTF-8 带 BOM**（`EF BB BF`） | 无 BOM + 含中文 → PS 5.1 按 GBK 解析 → **直接崩溃、exit 1、零输出**，极难排查 |
| `.vbs`（VBScript） | **ANSI / GB2312**（代码页 936） | 存成 UTF-8 → wscript 按 ANSI 读 → **中文注释乱码**，甚至语法错误 |

### 为什么 `.ps1` 必须带 BOM

Windows PowerShell **5.1** 在读取**无 BOM** 的 UTF-8 文件时，会按**本地 ANSI 代码页（中文系统 = GBK/936）**解析。
文件里有中文注释或中文字符串时，字节被错解 → 解析失败。

**症状极具迷惑性**：脚本**没有任何报错输出**，直接退出，退出码 1。
（对比：如果是有 BOM 的 UTF-8，5.1 能正确识别。）

补 BOM 的方法（写脚本后立刻执行）：

```powershell
$path = "路径\xxx.ps1"
$content = [System.IO.File]::ReadAllText($path)
$utf8bom = New-Object System.Text.UTF8Encoding($true)   # $true = 带 BOM
[System.IO.File]::WriteAllText($path, $content, $utf8bom)
```

### 为什么 `.vbs` 反过来要 GB2312

wscript 读 `.vbs` 时按 **ANSI（本地代码页）** 解释，不认 UTF-8 的 BOM。
所以 `.vbs` 里的中文必须存成 **GB2312（代码页 936）**，且**不带 BOM**。

```powershell
# 把 Write 工具刚写出的 UTF-8 内容，转存为 GB2312
$vb = "路径\run-hotspot.vbs"
$c  = [System.IO.File]::ReadAllText($vb, [System.Text.Encoding]::UTF8)
[System.IO.File]::WriteAllText($vb, $c, [System.Text.Encoding]::GetEncoding(936))
```

### 一句话铁律

> **`.ps1` → UTF-8 带 BOM；`.vbs` / `.bat` → ANSI(GB2312)。两者方向相反，改完必须验证。**
> 验证方式：`.ps1` 跑一次看有没有输出；`.vbs` 跑一次看 `wscript` 退出码是否为 0。

---

## 三、最隐蔽的坑：临时诊断脚本不带 BOM → 误判"文件不存在"

排查过程中，写了个临时脚本来 `Test-Path '中文名.bat'`，结果**报文件不存在**，一度以为生成的启动器丢了。

**真凶**：那个**临时诊断脚本自己**是 Write 工具写的 UTF-8 **无 BOM** 文件 →
里面的中文字面量 `'开启移动热点.bat'` 被 PS 5.1 按 GBK 解析 → **变成乱码路径** → `Test-Path` 当然找不到。

实际文件一直在，是"用来找文件的工具"自己坏了。

打印出来一看路径就露馅了：

```
launcher=C:\Users\Administrator\Desktop\寮€鍚Щ鍔ㄧ儹鐐?bat    ← 乱码
exists=False
```

### 一句话铁律

> **凡是含中文的 `.ps1`，哪怕是跑一次就删的临时诊断脚本，也必须带 UTF-8 BOM。**
> 否则你会被自己写的探针骗到，往错误方向排查半天。

---

## 四、日志策略：定时任务别"每次都写"

### 现象

v1.0 每次运行都写一行日志。5 分钟一次 = **一天 288 行**，一天就滚到 241 行，且**永远只增不减**。

### 做法：只在"状态真的变化"时记录

静默模式（计划任务所用的 `-Quiet`）下分三档：

| 情况 | 是否写日志 | 说明 |
|---|---|---|
| 目标本来就正常（热点已开） | **不写** | 绝大多数情况，占 99% |
| 发生实际动作（热点被关→脚本开回来） | **写一行 `[OK]`** | 真正需要留痕的事 |
| 出异常 | **写 `[ERROR]`** | 必须能被发现 |

实现上给日志函数加一个级别过滤 + 一个 `-Always` 强制开关：

```powershell
function Write-Log {
    param([string]$Message, [string]$Level = 'INFO', [switch]$Always)
    if ($Quiet -and -not $Always) {
        if ($Level -eq 'INFO' -or $Level -eq 'DEBUG') { return }   # 静默模式吞掉日常噪音
    }
    # ... 写文件
}
```

**"跳过"这条正常路径里不要再补一句 `Write-Log ... -Level 'OK'`**——那正是膨胀的来源。
（v1.0 就是在这里多写了一句 `完成：热点已开启`。）

### 实测效果

| 场景 | 日志行数 |
|---|---|
| 热点本已开启时运行 | 241 → **241**（零增长） |
| 热点关闭后运行（成功开启） | 241 → **242**（仅 +1） |

### 一句话铁律

> **定时任务 + 每次写日志 = 日志必然膨胀。只记录「状态变化」和「异常」。**

---

## 五、配套：WinRT 调用与计划任务的其他要点

本次工具用 WinRT 控制移动热点，顺带攒了几条：

| 项 | 事实 / 规则 |
|---|---|
| 运行环境 | **必须 Windows PowerShell 5.1**。PS 7（Core）加载不了该 WinRT 投影，报 `Unable to find type` |
| 异步等待 | `StartTetheringAsync()` 等返回 `IAsyncOperation`，PS 5.1 下**需手动 Await**（`WindowsRuntimeSystemExtensions.AsTask` 泛型反射）；但 `GetInternetConnectionProfile()` 是**自动解包**的，不能再套 Await |
| 计划任务账户 | 移动热点绑在**当前用户会话**，计划任务要用**当前登录用户 + `RunLevel=HighestAvailable`**，**不要用 SYSTEM**（会话 0 很可能操控不了） |
| 注册计划任务 | `schtasks /Create /XML` 会因 XML 声明（UTF-16）与实际文件编码（UTF-8）不符**静默失败**；改用原生 **`Register-ScheduledTask -Xml`** 一次成功 |
| 校验成败 | 别只看命令回显。用 `git ls-remote` 比 hash、`Get-ScheduledTaskInfo` 看 `LastTaskResult`、看报告文件里有没有期望的行 |
| 探测方式 | 沙箱对子进程 stdout 捕获不可靠（常返回空）→ **让脚本自己 `Out-File` 写文件，再读文件** |

---

## 六、一句话速查

1. 后台任务要真静默：**从创建进程起就无窗口**（VBS `Run(cmd,0,True)` / `CREATE_NO_WINDOW`），`-WindowStyle Hidden` 会闪。
2. `.ps1` 编码 = **UTF-8 带 BOM**；`.vbs`/`.bat` 编码 = **ANSI(GB2312)**。方向相反，别搞反。
3. 临时诊断脚本只要含中文，**也要带 BOM**，否则探针自己会骗你。
4. 定时任务日志**只在状态变化/异常时写**，否则一天几百行、永远膨胀。
5. 改完编码**必须验证**：`.ps1` 看有无输出，`.vbs` 看 `wscript` 退出码是否为 0。
6. 计划任务注册用 `Register-ScheduledTask -Xml`，别用 `schtasks /XML`。
7. 涉及当前用户会话的资源（如移动热点），计划任务账户用**当前用户 + HighestAvailable**，别用 SYSTEM。
8. 判断成败 grep 证据（hash / LastTaskResult / 日志行），**不看命令回显、不看退出码猜**。
9. 改功能 → 同步更新 **CHANGELOG + README 版本段**，不能只靠 commit message（本次就被用户追问过一次）。
10. 沙箱环境里探测一律"**脚本写文件 → 读文件**"，别依赖子进程 stdout。
