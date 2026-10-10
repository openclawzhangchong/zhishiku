# 09 · CPU 占用归零尝试（失败）与脚本安全改写

> **摘要**：两件事。其一，实测发现 `llama-server` 生成时 CPU 占约 **1 个核**（空闲为 0），怀疑是 `--poll`（CUDA 忙等轮询）所致，遂改成 `--poll 0` 验证 —— **假设被推翻**：CPU 只省 0.03 核，速度却从 **107.6 掉到 76.3 tok/s（-29%）**，已回滚。其二，改脚本过程中把 `start-llama-server.bat` **写成了 0 字节**，靠改前几分钟的备份救回。后者比前者更值得记住：一行 `open(P,"wb").write(...)` 就能清空文件。附可复用的 CPU/内存测量代码与操作检查清单。
>
> **关键词**：--poll；CUDA 忙等；CPU 占用；GetProcessTimes；WorkingSetSize；全 GPU 推理；open(wb) 截断；UnicodeEncodeError；bat 安全改写；备份

---

## 一、踩过的坑（吃过哪些亏）

1. **误判 CPU 占用的来源**：看到「CPU 占 1 核 + 配置了 `--poll 100`」就认定是轮询所致。**错了。** 改成 `--poll 0` 后 CPU 几乎没变，速度却掉了 29%。凭参数语义猜因果，代价是一次无用的调优 + 一次性能回退。
2. **`open(P, "wb").write(txt.encode("ascii"))` 会把文件清空**：`open(P,"wb")` 在参数求值**之前**就执行，文件当场被截断成 0 字节；随后 `encode("ascii")` 因注释里混入中文抛 `UnicodeEncodeError`，`write` 从未执行 —— **文件却已经空了。**
3. **纯 ASCII 的 bat 里，注释也不能写中文**：本次就是注释块踩的。该文件要求全 ASCII，中文注释会直接让编码失败。
4. **Python 字符串里的 Windows 路径反斜杠会转义引号**：`'set "MODEL=models\' + X` 直接 `SyntaxError`。硬写反斜杠必踩。
5. **把「CPU 占 1 核」误当成「内存参与计算」**：两者无关。判断权重是否在内存，要看**进程工作集**（实测 1849.7 MB，而模型权重需 17086 MB），不是看 CPU。

## 二、怎样才能成功（成功做法）

- **改脚本前必须先备份，且备份要在同一次操作里先做**：`cp xxx.bat archive/xxx.bat.bak-$(date +%Y%m%d-%H%M%S)-<原因>`。本次能救回来，纯粹因为备份是几分钟前做的。
- **先编码，再落盘，且先写临时文件校验**（正确写法见第四节）。永远不要写 `open(P,"w").write(可能抛异常的表达式)`。
- **判断「有没有内存参与计算」看四条**：真实命令行含 `-ngl 999` 且无 `--no-kv-offload`；显存账对得上；**进程工作集远小于模型体积**；GPU 利用率稳定高位。
- **判断 CPU 是否在关键路径上看 GPU 利用率**：如果 CPU 真参与计算，GPU 会因为等 CPU 而掉利用率。实测 GPU 全程稳定 **69~82%** → CPU 不在关键路径上。
- **改参数前后各测一次速度**（用固定长度的生成任务），别凭感觉 —— 否则会把 29% 的回退当成噪声。

## 三、易失败点（哪些地方是坑 / 什么容易失败）

- **`--poll` 调不掉那 1 核**。实测对比（同模型、同 2500 token 任务、同 `-t 6`）：

  | 配置 | 速度 | 生成期 CPU | 空闲 CPU |
  |---|---|---|---|
  | `--poll 100` | **107.6 tok/s** | 0.93 核 | 0.00 核 |
  | `--poll 0` | **76.3 tok/s** | 0.96 核 | 0.00 核 |

  **CPU 省 0.03 核，速度亏 29%。** 结论：**不要为了省 CPU 去动 `--poll`。**
  官方说明：`--poll <0...100> use polling level to wait for work (0 - no polling, default: 50)`；
  `--poll-draft` 默认「same as `--poll`」，所以改 `--poll` 会连带覆盖 MTP 草稿路径。

- **那 ~1 核的真正来源（属推测，未逐一验证，不要当结论用）**：① llama.cpp 的 CPU 线程池在同步屏障上空转（`-t 6` 起 6 个线程，即使无 CPU 层也要参与每 token 的 barrier 同步）；② 采样 / 分词 / 模板处理；③ 每 token 的图构建（配合 `--spec-type draft-mtp` 投机解码会累积）；④ HTTP 层与 JSON/SSE 序列化（107 tok/s 即每秒 107 次响应组装）。
  **唯一实测确认的是「不是 `--poll` 造成的」。**

- **约 1 核 CPU 占用本身不是问题**：多个核心中的不到 1 个，不是瓶颈；速度 107.6 tok/s 与纯 GPU 推理相符（CPU 回退时实测仅 0.745 tok/s，差约 140 倍）。

- **不要从 Bash 里调 PowerShell**（会被安全策略拦），PowerShell 操作请用 PowerShell 工具；`tasklist` 之类取 PID 的操作用 Bash 直接跑即可。

## 四、具体怎么实现（实现方式）

**1. 安全改写文件的标准写法**

```python
data = txt.encode("ascii")                       # 1) 先编码: 抛异常也不伤原文件
assert len(data) > 1000, "结果异常短"
assert sum(1 for b in data if b > 127) == 0, "含非 ASCII"
open(TMP, "wb").write(data)                      # 2) 写临时文件
assert open(TMP, "rb").read() == data, "校验失败"
shutil.move(TMP, P)                              # 3) 最后才覆盖原文件
```

**Windows 路径拼接用 `BS = chr(92)`**，别硬写反斜杠：

```python
BS = chr(92)
'set "MODEL=models' + BS + MODEL_NAME + '"'
```

**2. 探测并保持原文件的换行风格**

```python
EOL = "\r\n" if txt.count("\n") == txt.count("\r\n") else "\n"
```
（本次目标文件是 **LF**，不是 CRLF，盲目转成 CRLF 属于额外风险。）

**3. CPU 占用测量（纯标准库，Windows）**

```python
import ctypes, ctypes.wintypes as w, time
K = ctypes.windll.kernel32
h = K.OpenProcess(0x0400 | 0x0010, False, PID)   # QUERY_INFORMATION | VM_READ

def cpu_seconds():
    c, e, kt, ut = (w.FILETIME(), w.FILETIME(), w.FILETIME(), w.FILETIME())
    K.GetProcessTimes(h, ctypes.byref(c), ctypes.byref(e),
                         ctypes.byref(kt), ctypes.byref(ut))
    f = lambda x: (x.dwHighDateTime << 32 | x.dwLowDateTime) / 1e7
    return f(kt) + f(ut)

a = cpu_seconds(); time.sleep(10); b = cpu_seconds()
print("平均占用 %.2f 个核" % ((b - a) / 10))
```

**判读**：增长 ≈ 采样秒数 → 1 核满载；≈ 采样秒数 × 6 → 6 线程全忙（配合 `-t 6` 即真在算）；≈ 0 → 空闲。

**4. 进程内存测量（判断权重是否在内存里）**

```python
class PMC(ctypes.Structure):
    _fields_ = [("cb", w.DWORD), ("PageFaultCount", w.DWORD),
                ("PeakWorkingSetSize", ctypes.c_size_t), ("WorkingSetSize", ctypes.c_size_t),
                ("QuotaPeakPagedPoolUsage", ctypes.c_size_t), ("QuotaPagedPoolUsage", ctypes.c_size_t),
                ("QuotaPeakNonPagedPoolUsage", ctypes.c_size_t), ("QuotaNonPagedPoolUsage", ctypes.c_size_t),
                ("PagefileUsage", ctypes.c_size_t), ("PeakPagefileUsage", ctypes.c_size_t),
                ("PrivateUsage", ctypes.c_size_t)]
m = PMC(); m.cb = ctypes.sizeof(m)
ctypes.windll.psapi.GetProcessMemoryInfo(h, ctypes.byref(m), m.cb)
print(m.WorkingSetSize / 1024 / 1024, "MB")
```

本次实测：工作集 **1849.7 MB** / 私有内存 3092.1 MB，而模型权重需 **17086 MB** → 权重只能在显存。

## 五、本次操作的可复用检查清单

改 `start-llama-server.bat` 这类**纯 ASCII + LF 行尾**的批处理时：

- [ ] **先备份**，文件名带时间戳与原因
- [ ] 读文件时**探测行尾风格**，保持原样
- [ ] 新增内容**全 ASCII**（含注释）
- [ ] 先 `encode()` 校验 → 写临时文件 → `shutil.move`
- [ ] 改完回读校验：字节数、非 ASCII 字节数、关键变量值
- [ ] 重启：`taskkill /f /im llama-server.exe` → `schtasks /run /tn "llamacpp-server"`
- [ ] 等就绪：加载约 50~60 秒，期间 503 属正常
- [ ] **验证「当前跑的是什么」看 `/props` 的 `model_path` + 系统里的真实命令行**（`Get-CimInstance Win32_Process`），不要相信 bat 的内容
- [ ] **性能回归**：改参数前后各测一次速度

---

> **一句话**：CPU 占约 1 核是 llama.cpp 的固有开销（线程池同步 + 采样/分词），
> 不是内存参与计算，也不是 `--poll` 造成的 —— 动 `--poll` 只会亏速度。
> 判断「有没有在算」看 GPU 利用率（稳定 70%+ 就说明 CPU 不在关键路径上）。
> 而改脚本时：**先备份、先编码、再落盘** —— 否则一行 `open(P,"wb")` 就能把文件清空。
