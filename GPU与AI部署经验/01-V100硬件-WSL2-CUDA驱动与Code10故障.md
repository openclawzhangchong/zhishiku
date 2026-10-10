# 01 · V100 硬件 / WSL2 / CUDA 驱动与 Code 10 故障排查

> **摘要**：记录 Tesla V100-PCIE-32GB（华为 OEM）在 ASUS H610M-K D4 消费级主板上，于 Windows 10 + WSL2 环境部署 CUDA 训练/推理的完整路径与最严重故障（设备管理器 Code 10 / `STATUS_INSUFFICIENT_RESOURCES`）的真实根因与锁定解法。结论均带证据，目标是装卡、装驱动、跑 CUDA 时直接复用，不再重蹈一整天的排查覆辙。
>
> **关键词**：Tesla V100；H610M-K D4；WSL2；CUDA 12.4；Data Center Driver；Code 10；MMIO；Above 4G Decoding；BAR1；nvidia-smi；PCIe；TCC 模式

---

## 一、踩过的坑（吃过哪些亏）

1. **Code 10 排查耗时一整天**：`nvidia-smi` 报 `exit 9`、设备管理器问题码 10、状态 `0xC000009A`（`STATUS_INSUFFICIENT_RESOURCES`）。早期误判为“驱动没装好”或“资源冲突”，反复重装驱动、PnP 重置，均只能热态假修复，**重启 100% 复现**。
2. **“设备 Status=OK” ≠ “驱动可用”**：仅靠 `Get-PnpDevice` 看状态 OK 会误判，必须同时看 `nvlddmkm` 服务状态 + `nvidia-smi` 退出码。
3. **热态修复 ≠ 冷启动可用**：PnP 重装后设备转 OK 只是热态假象，必须重启验证。
4. **只看“资源条数”会误判**：`Win32_PnPAllocatedResource` 不区分 BAR0/1/3，必须逐条解析 `Antecedent` 的 `StartingAddress` 对照 BAR 用途。
5. **驱动级显存泄漏**：被强杀的 python 进程留下 ~26GB 幽灵显存占用，Windows 性能计数器看不到持有者，webui 加载 CUDA OOM，NVML `Unknown Error`，`nvidia-smi` 全程不可用。软重置设备状态 OK 但 CUDA/NVML 未恢复。
6. **NVIDIA 直链程序化下载一律 403**：`us/cn/international.download.nvidia.com` 对脚本下载返回 403，驱动必须用浏览器手动下载。
7. **物理供电没插紧**：V100 Code 10 曾因 6+2pin 中 2pin 未插紧导致。
8. **WSL Ubuntu Appx 静默失败**：26.04 的 `ubuntu.exe install --root` 静默失败，rootfs 文件名写错（`-wsl.rootfs.tar.gz` 而非 `-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz`）也导致导入失败。
9. **WSL 两个功能会被安全中心黑名单拦截** `wsl.exe`：需在用户会话手动放行（放行下载路径无效）。

## 二、怎样才能成功（成功做法）

- **锁定工作组合，不要再动**：Data Center Driver **551.61（CUDA 12.4）** + BIOS **Above 4G Decoding=Enabled**。9/11 重装 551.61 后官方 build 10853（CUDA 12.4 编译）直接三档全通。
- **WSL2 + Ubuntu 22.04 成功态**：systemd + `torch 2.6.0+cu124`，GPU 验证 `is_available=True` / `Tesla V100-PCIE-32GB` / `sm(7,0)`。
- **显存泄漏可靠解法**：Windows 上驱动级泄漏 + NVML 损坏，唯一可靠解法是**重启系统**。
- **V100 重启后无法启动的首选处置**：Tesla 不支持 Zero Power，软重启不复位 → **完全关机 + 拔电源线 30 秒以上再开机**；其后才是 BIOS（Above 4G Decoding、试关 Re-Size BAR 让卡退回 256MB small BAR1），最后 DDU/驱动。
- **驱动安装**：数据中心驱动选 **R560/R565，避开 R570+**；WSL2 内只装 CUDA toolkit 不装驱动；框架锁 12.x。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **Code 10 真根因 = BIOS 4G+ PCIe MMIO 窗口不足**：V100-32GB 规格 BAR0=16MB / BAR1=32GB / BAR3=32MB，需要 ≥32GB 的 4G+ 连续窗口；本机 ≥4GiB 区间仅 316.1MB（3 簇），需求:供给 ≈ 104:1。同类案例：V100-32GB `DEV_1DB5` 旧驱动 462.30 可用、551.78/582.70 失败。
- **R580 是最后一个支持 pre-CC7.5（含 Volta 7.0）的分支**；官方建议 pre-CC7.5 用户“驱动停 580 + CUDA Toolkit 停 12.9”。
- **vLLM / SGLang / Fish S2 Pro 对 V100 均不可用**（需 sm_75+）；LLM 推理统一用 **llama.cpp**。
- **Tesla 卡是 TCC 模式、数据中心驱动**，WSL2 官方硬性不支持切 WDDM。
- **中文 Windows 无驱动显卡叫“3D 视频控制器”**（非英文名）→ 检测用 `Get-CimInstance Win32_PnPEntity | Where-Object { $_.Name -like '*VEN_10DE*' }`。错误码 28 = 驱动未装；错误码 10 = 启动失败。

## 四、具体怎么实现（实现方式）

**1. BIOS 行动清单（若再 Code 10）**：Advanced → PCI Subsystem Settings：
`Above 4G Decoding=Enabled`（关键）、`Re-Size BAR Support` 关/开都试（关掉退回 256MB 小 BAR1，恰有一块整 256MB 窗口承接）、`DVMT Pre-Allocated` 最小、`PCIe Speed=Gen3`、`Primary Display=PEG` + `iGPU Multi-Monitor=Enabled`。若 BIOS 隐藏该项 → modGRUBShell `setup_var` 改隐藏变量，或换 B660/Z690。

**2. WSL2 部署关键步骤**：
- `.wslconfig` 已存在且正确（12GB / 6 核）→ 先备份再改。
- 用 `wsl --import` + cloud-images rootfs（文件名 `ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz`）。
- 建用户（sudo NOPASSWD，密码 `V100@2026`）、`build-essential`/`python3-pip`/`python3-venv`、`/etc/wsl.conf`（systemd）。
- `venv` 复制 exe 会被沙箱静默拦截 → 用 `pip --user` 装框架。

**3. PnP 修复命令（管理员 PowerShell）**：
```powershell
Select-String -Path C:\Windows\INF\setupapi.dev.log -Pattern 'not started: Device has problem'
pnputil /export-driver oem12.inf /destination:d:\driver-backup
pnputil /remove-device "PCI\VEN_10DE&DEV_1DB6&SUBSYS_124A10DE\..."
pnputil /scan-devices
pnputil /add-driver "D:\llamacpp\driver-backup\nv_dispwi.inf" /install
pnputil /restart-device "PCI\..."
```
注意 `pnputil` 无 `/properties`；`$PID` 是 PS 只读变量别当循环变量。重启后仍 Code 10 → 走 BIOS 排查 / 换旧驱动（462.30）/ DDU 清装 / 物理重插（供电务必插紧）。

**4. 重启后无法启动快速诊断**：
`Get-PnpDeviceProperty -KeyName DEVPKEY_Device_ProblemStatus` ⇒ `3221225626 = 0xC000009A`；`ProblemCode=10`；`DevNodeStatus=0x01802400`（有 `DN_HAS_PROBLEM`，无 `DN_DRIVER_LOADED`/`DN_STARTED`）。判据作废项：`LogConf\BootConfig` 无效；`setupapi.dev.log` 搜 `not started` 可能 0 命中（日志滚动）→ 一律以 DEVPKEY 为准。

> 关联技能：`windows-gpu-pcie-bar-diagnosis`（已按本次实测更新）。
