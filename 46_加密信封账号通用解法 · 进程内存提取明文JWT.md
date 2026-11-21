# 46 · 加密信封账号的通用解法：客户端进程内存里就有明文

> **摘要**：官方客户端把 token 加密落盘（$wbEncrypted 信封）后，外部工具"导入即失效"——密文解不开、刷不了。破解加密两条路都走不通（运行时密钥不落盘；原生模块只导出 createTSObject 工厂）。但**客户端进程内存里存着解密后的明文 JWT**（它自己要用）——扫描进程内存按 JWT payload 的 sub=uid 匹配，就能提取明文回填。比破解加密简单两个数量级。

**关键词**：wbEncrypted, envelope, 内存扫描, ReadProcessMemory, VirtualQueryEx, JWT, Keycloak, sub, Bearer, Offline, 明文凭据, 导入失效, 进程内存提取, 加密存储

---

## 主题 1：信封账号"导入即失效"——先别想破解，看客户端自己的明文

| 项 | 内容 |
|---|---|
| **症状** | 从客户端导入的账号显示「该账号凭据为加密存储，仅支持切换账号」；反代 503 |
| **排查过的死路** | ① 运行时密钥不落盘：bootstrap（symmetricKey.keyDerBase64）由主进程原生层（编译进 exe 的 storage 桥）内存提供，经 IPC（daemon:credential-protection-bootstrap）推给 daemon，磁盘上只有被它加密的 keyblob；② 原生模块只导出 createTSObject 工厂（vtable 不公开） |
| **活路** | 客户端 daemon 进程内存里存着**解密后的明文 JWT**（typ=Bearer 的 access、typ=Offline 的 refresh，payload.sub=uid 可精确匹配） |
| **修复** | 扫描进程内存（OpenProcess+VirtualQueryEx+ReadProcessMemory，同用户进程即可）→ JWT 正则 → 解 payload 按 sub 匹配 → 回填账号库。实测双渠道闭环：提取→回填→反代对话 200 |
| **可复用教训** | **加密落盘 ≠ 拿不到明文**。客户端运行时必然持有明文（它自己要用），先扫进程内存再考虑逆向。搜索特征：Keycloak JWT 三段式 `eyJ…eyJ…`，payload 的 `sub` 对账号、`typ` 对用途（Bearer/Offline） |

**验证状态**：真机实测（国内 uid 2b0499fb + 国际 uid d5bfe271，提取→回填→反代 200）

## 主题 2：进程内存扫描的实现要点（Windows）

| 项 | 要点 |
|---|---|
| 权限 | 同用户进程即可：`OpenProcess(PROCESS_VM_READ(0x10)\|PROCESS_QUERY_INFORMATION(0x400))` |
| 枚举 | `VirtualQueryEx` 逐区域推进；只读 `MEM_COMMIT && PAGE_READWRITE` 的区域；单区域 >64MB 跳过（图片/映射区）；总量上限 ~800MB 防失控 |
| 扫描 | 手写 `eyJ…eyJ…` 三段式扫描（不引 regex 依赖）：`is_ascii_alphanumeric() || '_' '-' '='` 为合法字符 |
| 过滤 | 解 payload（base64url 无 pad）按 `sub` 匹配 uid；`typ=Bearer`→access、`typ=Offline`→refresh |
| 集成 | 导入流程自动触发：导入后凭据是信封 && 客户端进程在跑 → 提取回填；失败保持信封原样（零影响） |
| 坑 | JWT 正则第三段没有终止符，长度阈值要卡准（>40 字符）防误匹配；`claims` 解码失败要 try 跳过（内存里有半截 token） |

**验证状态**：真机实测（Python 原型 + Rust 集成双通过）

## 主题 3：对比表 —— 三种"客户端加密凭据"的处理路线

| 场景 | 能不能拿到明文 | 路线 |
|---|---|---|
| 客户端**在本机运行着** | ✅ 能 | 扫描进程内存提取（本篇） |
| 客户端**装了但没开** | ⚠️ 启动客户端即可 | 先启动再扫描；或走 OAuth 重新登录 |
| 客户端**没装** | ❌ 内存都没有 | 走 OAuth 拿明文（`tools/oauth-login.py`）；或破解 at-rest 加密（不可行，密钥运行时注入） |

**排查顺序建议**：`tools/check-credentials.py` 看形态 → ENVELOPE 则看客户端是否在跑 → 在跑就内存提取 / 没跑就启动后再提取 / 不想装客户端就 OAuth。

> 关联：`42_Electron加密凭据与多渠道伪装`（加密机制全链路）、`44_设备令牌不破解也拿到`。
