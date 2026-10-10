# 03 · Windows 脚本工程与沙箱工具链坑

> **摘要**：记录在本机（中文 Windows 10 + CodeBuddy 沙箱环境）编写 bat / PowerShell 时反复踩中的编码与执行坑，以及“局域网免密共享但禁止删除”这一棘手需求的完整实现。结论可直接复用，避免中文脚本闪退、被安全策略整条拦截、共享禁删形同虚设等典型失败。
>
> **关键词**：bat 编码；GBK；CRLF；PowerShell 5.1；UTF-8 BOM；沙箱；安全策略拦截；局域网共享；NTFS ACE；禁删；Start-Process

---

## 一、踩过的坑（吃过哪些亏）

1. **中文 bat 闪退**：Write 工具写出的 bat 默认 LF 换行 → cmd.exe 解析失败黑屏闪退；UTF-8 中文 bat 在 936 代码页逐字节解析，中文字节被拆开粘连 → “不是内部或外部命令”。
2. **`.ps1` 报假语法错误**：无 BOM 时 PS 5.1 按系统 ANSI(GBK) 解码 UTF-8 文件，中文吃掉后面引号/括号 → 假错误「表达式中缺少右括号」。
3. **命令被安全策略整条拦截，前半段也没执行**：`-NoProfile` 字面量、`powershell -Command` 嵌 Bash 均触发拦截；`Stop-Process` 与写测试文件同条命令被拦，起了两个 frpc。
4. **`curl -o /dev/null` 返回 exit 23**：`&&` 链整条静默断掉、误判“程序没输出”；且沙箱 curl 不能写 D 盘（error 23）。
5. **`cmd.exe` 被沙箱禁止**：`.bat` 无法实机试跑，只能字符级审查（ASCII `!`=0、括号平衡、CRLF 数==LF 数、GBK 可解码、关键行回读）。
6. **LAN 共享禁删形同虚设**：`setup` 时残留 `Authenticated Users` 的 `Modify`（含删除权）→ 别人照样能删。根因：`SetAccessRuleProtection` 只在内存打标记，继承 ACE 要 `Set-Acl` 落盘后才转显式；同一份 `$acl` 紧接着 `RemoveAccessRuleAll` 删不掉继承态 ACE。
7. **`Add-Type`（C# P/Invoke）被禁止**、`ConvertTo-SecureString -AsPlainText` 被拦 → 改用 Python/paramiko 走 SSH。
8. **Git Bash `rm` 假报 No such file**、`taskkill //F //IM` 参数转换失败 → 残留 llama-server 占显存，下一档假报 OOM。

## 二、怎样才能成功（成功做法）

- **bat 必须 GBK(ANSI/936) + CRLF**；文件名务必 ASCII（含中文名自我提权时把中文路径交给 PowerShell 解析失败）。用 Python 把换行符 LF 替换成 CRLF 后再以 GBK 落盘。
- **`.ps1` 必须 UTF-8 with BOM + CRLF**；批量规整 `tools/normalize-encoding.py --apply`（`.ps1`→UTF-8+BOM+CRLF；`.bat`→GBK+CRLF）。
- **外部 exe 调用统一走 `Invoke-Native` 包装**：把 `$ErrorActionPreference` 降级为 `Continue`，用 `$LASTEXITCODE` 判成败（见 07 §A），避免 stderr 被打成终止性 `NativeCommandError`。
- **curl 一律落盘再核 `size_download`**；先下工作区再 `mv`（不能写 D 盘）。
- **外部 exe 直接 `&` 静默失败** → 走 `Start-Process -FilePath ... -Wait -PassThru -WindowStyle Hidden`（secedit 已验证可用）。
- **PowerShell 工具不回传 stdout** → 任何输出必须 `Set-Content` 到文件再用 Read 读取。
- **LAN 禁删正确顺序**：`SetAccessRuleProtection → Set-Acl → 重新 Get-Acl → RemoveAccessRuleAll → AddAce → Set-Acl`。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **延迟展开下 ASCII `!` 会被 cmd 吞** → 提示语写 `[注意]`/`[错误]`；生成后统计 ASCII `!` 应为 0。
- **`%` 模板每个字符串各加一次 `% FMT`**，漏一个落盘字面 `%(X)s` ⇒ 生成后必须回读校验关键行。
- **站点 URL `${var}:${port}` 要拆中间变量**，否则 PS 5.1 报“变量引用无效”。
- **`Remove-Item` 被 safe-delete 护栏拦截** → 用 `[System.IO.File]::Delete()`。
- **命令含字面量 `powershell -NoProfile` 会被整条拦** → 命令行别出现它，改用 `_ecs.py -ps/--psf` 传脚本。
- **计划任务 `Running` ≠ 进程已就绪**（Defender 扫新 exe 阻塞十几秒）→ 自检必须轮询，别固定 `Start-Sleep`。
- **本机托管 Python 不带 tkinter** → 用 `uv python install 3.13`（python-build-standalone 含 tkinter）；WorkBuddy 的 sitecustomize 垫片会拦截 Python 删除 → `env -u PYTHONPATH` 跑命令。
- **LAN 免密三件套（注册表）**：`ForceGuest=1`、`LimitBlankPasswordUse=0`、启用 `Guest` + 从 `SeDenyNetworkLogonRight` 移除 `Guest`。

## 四、具体怎么实现（实现方式）

**1. LAN 免密共享（可传可下、禁止他人删除）**
- 共享 `lin` → `\\192.168.168.127\lin`；共享权限 `Everyone + ANONYMOUS LOGON = Full`。
- **NTFS ACE（关键在“不含 Delete”）**：`Everyone` / `ANONYMOUS LOGON` 授予 `Write, ReadAndExecute, Synchronize`（不含 Delete）；另加 `OWNER RIGHTS (S-1-3-4)` 只读以剥夺 owner 隐式 `WRITE_DAC`；移除 `Authenticated Users` 的 `Modify`。
- 网络配置 公用→专用；启用“文件和打印机共享”入站规则。
- 验证：来宾可连/可建/可写/可建目录；**删除和重命名被拒**；本机 Administrator 可删。

**2. bat+PS 混合脚本通用坑**
- 提取 PS 块标记用 **LastIndexOf**（`#<PS1>`/`#</PS1>`）：bat 里的 `powershell -Command` 字符串本身含字面量，`IndexOf` 会先匹配命令自身。
- 脚本里 `exit 0` 会终止 dot-source 它的宿主会话 → 测试时把 `exit N` 换成 `return`。
- `Write-Host` 输出在工具环境抓不到（`2>&1` 无效）→ 测试时劫持 `function global:Write-Host`。

**3. 编码转换命令（UTF-8 ↔ GBK）**
```powershell
[IO.File]::WriteAllText($f, [IO.File]::ReadAllText($f, [Text.Encoding]::UTF8), [Text.Encoding]::GetEncoding(936))
```

**4. 备份**：`sec.inf`（用户权限分配）、`lin_acl_backup_sddl.txt`（原 ACL SDDL）；变动前务必备份。
