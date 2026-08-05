---
name: mcu-serial-to-mfc-dll
description: "把单片机串口(UART/RS232)文本协议封装成 Windows MFC 动态库(DLL)：VS2010 工程、必须生成 .sln、全部源码 UTF-8 带 BOM、含 Win32 与 x64 双平台配置、正确处理 MFC 链接模式。Use when the user says 转动态库/生成DLL/封装成库/串口协议转DLL/把命令做成DLL, or asks for 生成sln/UTF-8带BOM/64位/MFC库. Do NOT use for writing EXEs or plain static .lib."
---

# 单片机串口协议 → MFC 动态库 (DLL)

把单片机（STM32 等）上的串口通信协议封装成 Windows 动态库，供 Python ctypes / LabVIEW / C# / MFC 等主机程序调用。主机每次调用一个导出函数即完成一次"组帧→发命令→收应答→解析"的完整事务。

标准参考实现：`D:\projects_xzk\FLC1015gongzhuang\FLC1015\FLC1015DLL\`（VS2010，18 条导出命令，Win32/x64 双平台，是复制改名的模板）。

## When to use

用户让把一块单片机固件里的串口命令集封装成 DLL。典型触发："封装成库"、"转动态库"、"生成DLL"、"串口的 DLL"、"要求生成 sln / UTF-8带BOM / 64位 / MFC库"。

## 硬性约定（勿违背）

| 项 | 约定 | 原因 |
|----|------|------|
| VS 版本 | Visual Studio 2010（vcxproj Format Version 11.00） | 用户指定 |
| .sln | 必须生成，且其中 ProjectGuid 与 vcxproj 完全一致 | 用户要求；15 个工程文件里 .sln 最容易被漏掉 |
| 文本编码 | 所有 .h/.cpp/.def/.txt 一律 **UTF-8 带 BOM**（BOM= EF BB BF） | VS2010 按 BOM 识别编码 → 中文注释任何区域设置不乱码；GBK 无 BOM 在外面代码页环境会花屏。禁止交付 GBK/无 BOM 版本 |
| 平台配置 | 4 个工程配置：Debug/Release × Win32/x64 | 必须同时存在，缺 x64 = 没有 64 位支持 |
| MFC 链接 | Win32：`UseOfMfc=Static` + runtime `MultiThreaded(Debug)`；x64：`UseOfMfc=Dynamic` + runtime `MultiThreadedDLL(Debug)` | **VS2010 的 x64 平台没有静态 MFC 库**（nafxcwd.lib 只有 32 位）。x64 设 Static 会错链 32 位 MFC，报 LNK2001 __argv/__argc + LNK1120。要 x64 静态 MFC 必须 VS2013 及以上 |
| DllMain | 绝对不要自定义 DllMain | MFC 库（静态/动态链接都）自带 DllMain，自定义产生 LNK2005 重复定义 |
| 互斥锁 | 串口临界区用**静态全局对象**（构造函数 InitializeCriticalSection，析构 DeleteCriticalSection） | 消除对 DllMain 的依赖，Static/Dynamic 两种 MFC 模式都兼容 |
| 导出方式 | 只通过 .def 的 EXPORTS 段导出；头文件**禁止**再写 __declspec(dllexport) | 双源导出产生 LNK4197 重复导出警告 |
| .def 内容 | LIBRARY + EXPORTS 清单，**不写 DESCRIPTION 行** | x64 平台不支持 DESCRIPTION → LNK4017 |
| 调用约定 | 全部导出函数 `extern "C"` + `__stdcall` | 32 位函数名加下划线前缀 `_SetHgState@12`，64 位无修饰，dumpbin 可验证 |
| 波特率 | 与单片机 USART 一致（FLC1015 为 9600, 8N1） | 不匹配则无应答 |

## 工程结构与职责（按 FLC1015DLL 模板）

| 文件 | 职责 |
|------|------|
| `StdAfx.h/.cpp` | 预编译头；include 全部 MFC 头（afxwin/afxext/afxdisp/afxcmn/afxdlgs）；vcxproj 中 PrecompiledHeader=Create |
| `*Serial.h/.cpp` | 串口底层：BuildPortName（COM10+ 自动加 `\\.\` 前缀）、CreateFile 打开、PurgeComm、WriteFile + **循环 ReadFile** 直到收满 respLen 字节或超时、全局临界区锁串行化事务 |
| `*Comm.h/.cpp` | 协议层：每条命令组帧（文本帧 `cmd_xxx:param\r\n`）、计算应答长度（通常 10 字节 ASCII '0'/'1'）、解析应答、按命令给独立超时 |
| `*Interface.h/.cpp` | 导出层：每个命令一个导出函数，与 .def 一一对应 |
| `<名>DLL.h/.cpp` | MFC 入口：CWinApp 子类 + 全局 theApp + 空消息映射，**无 DllMain** |
| `*.def` | `LIBRARY "<名>"` + 全部导出函数名清单 |
| `ReadMe.txt` | Python ctypes 调用示例、x64 版需 VC2010 x64 运行库说明 |
| `.sln/.vcxproj/.vcxproj.filters` | 工程文件 |

接口统一签名约定：
- 每个函数返回 1=成功、0=失败
- 端口名经 `char*` 传入（如 "COM1"、"COM12"）
- 输出结果通过尾部指针参数带回（如 `GetDevicePC(char*, int, int* pCurrent)`）
- GPIO 数组参数用 10 个 int，每路取值 0/1；读取结果同样用 10 int 数组带回

## Procedure

### Step 1: 分析固件串口协议

1. 找到固件串口处理文件（FLC1015 为 `HARDWARE\RS232\RS232.c`，命令是 `cmd_xxx` 分支）。
2. 逐条列出：命令名、参数、**应答字节数与格式**、超时特性。
   - 普通命令：整体超时约 1000ms
   - 需单片机等待外部应答的（如 cmd_tri 等 PA3 应答最长 4s）：超时放大到 6000ms
3. 记录特殊约定：如 cmd_rsm 需在帧尾额外补 0x01 字节；cmd_232 正常应答特征为 `00 05 00 05` + 30 2D 前导；cmd_epc 电流 >=1000mA 返回 0xFF×3 哨兵。
4. 整理一张"命令→组帧→应答长度→超时"对照表，这是写代码的唯一依据。

### Step 2: 创建工程文件（sln + vcxproj）

最快且最可靠：**直接复制 FLC1015DLL 整个目录作模板**，然后：
1. 重命名为新工程名，重写所有 `<FLC1015xxx>` 符号为新名字。
2. vcxproj 中只改：工程名、ClCompile 项的文件名、ProjectGuid（**必须重新生成一个新 GUID**，且 sln 里要一致）。
3. 4 个工程配置块保持：Debug/Release × Win32/x64。
   - Debug|Win32 / Release|Win32：`UseOfMfc>Static`，RuntimeLibrary `MultiThreadedDebug` / `MultiThreaded`
   - Debug|x64 / Release|x64：`UseOfMfc>Dynamic`，RuntimeLibrary `MultiThreadedDebugDLL` / `MultiThreadedDLL`
   - 调用的框架"/中，配置属性→常规→MFC的使用"路径。
4. .sln 用 VS2010（Format Version 12.00）文本格式，GlobalSection(ProjectConfigurationPlatforms) 需列全 4 种工程配置。

如果不允许复制模板，也可手写 vcxproj，但注意 VS2010 的 vcxproj 字段（ConfigurationType、UseOfMfc、CharacterSet=MultiByte、PrecompiledHeader、OutDir/IntDir 按平台分目录 `.\Debug\` vs `.\x64\Debug\`）。

### Step 3: 编写源码（要求 UTF-8 带 BOM）

1. 所有新文件用 write 写成 UTF-8。write 工具默认输出的 UTF-8 **不带 BOM**，写入后需强制补 BOM（见 Step 4）。
2. 串口底层注意：
   - BuildPortName：单个字符的 `COMx` 会被 Win32 解释为设备名而非串口名，一律加 `\\.\` 前缀，兼容已带前缀的输入。
   - 读应答**必须循环 ReadFile**：串口分片到达，一次 ReadFile 常读不满；循环读直到收满 respLen 字节或整体超时。这是老版 DLL 每条命令发两次的根本原因。
   - 发送前 PurgeComm 清接收缓冲，防残留数据干扰。
3. 导出函数写法（Interface 层）：
   ```cpp
   extern "C" unsigned int __stdcall SetHgState(char *sComPort, int iBaudRate, int iState)
   {
       // 组帧 -> FLC1015_SerialExec -> 解析应答 -> 返回 1/0
   }
   ```
   iBaudRate 内部强制为 9600，与单片机一致。
4. 互：锁定义在 Serial.cpp 顶部，全局静态对象：
   ```cpp
   class CSerialLock {
   public:
       CSerialLock()  { InitializeCriticalSection(&m_cs); }
       ~CSerialLock() { DeleteCriticalSection(&m_cs); }
       CRITICAL_SECTION m_cs;
   };
   static CSerialLock g_serialLock;
   ```
5. .def 文件（导出完全由此控制）：
   ```
   LIBRARY   "FLC1015DLL"
   EXPORTS
     SetHgState
     SetShutterState
     ...（全部导出函数）
   ```
   注意：无 DESCRIPTION 行；lib 文件名与 DLL 名一致。

### Step 4: 保证编码为 UTF-8 带 BOM

写完或从旧工程复制后，统一做一次"无 BOM → 带 BOM"转换并逐文件验证：

```powershell
# 源文件是 GBK/GB2312/ANSI(936) 时：
$src = [System.Text.Encoding]::GetEncoding(936)
# 源文件已是无 BOM 的 UTF-8 时改用：$src = [System.Text.Encoding]::UTF8
$dst = New-Object System.Text.UTF8Encoding($true)   # 带 BOM
Get-ChildItem *.h,*.cpp,*.def,*.txt,*.sln,*.vcxproj | ForEach-Object {
    $c = [System.IO.File]::ReadAllText($_.FullName, $src)
    [System.IO.File]::WriteAllText($_.FullName, $c, $dst)
}
```
验证（每个文件都要过）：
```powershell
$b = [System.IO.File]::ReadAllBytes('FLC1015Serial.cpp')
'{0:X2} {1:X2} {2:X2}' -f $b[0],$b[1],$b[2]   # 期望输出 EF BB BF
```
**rule**：绝不要为了兼容旧编译器（如本机 VC6）把正式文件转回 GBK——VC6 不认 UTF-8 BOM（C2018 unknown character 0xef），需要做语法验证时只把"副本"转 GBK 放到临时目录编译，正式目录保持 BOM。

### Step 5: 构建与验证

1. 在 VS2010 打开 .sln，选择目标配置生成。
   - 32 位用 `Release|Win32`（`.\Release\xxx.dll`）；64 位用 `Release|x64`（`.\x64\Release\xxx.dll`）。
2. 检查导出符号：
   ```
   dumpbin /EXPORTS .\x64\Release\xxx.dll
   ```
   导出函数名应与 .def 完全一致、无修饰。
3. 用 Python ctypes 实测（ReadMe 有参考）：
   ```python
   from ctypes import *
   dll = CDLL(r'.\x64\Release\xxx.dll')
   dll.SetHgState.argtypes = [c_char_p, c_int, c_int]
   dll.SetHgState.restype  = c_uint
   print(dll.SetHgState(b'COM3', 9600, 1))
   ```
4. 语法验证（本机只有 VC6 时）：把 .h/.cpp 副本转 GBK 放临时目录，用 VC6 `cl /c` 编译 5 个 cpp 检查 0 错误（仅作语法检查，不替代 VS2010 的真实构建）。

## Troubleshooting（常见错误对照）

| 报错 | 原因 | 处理 |
|------|------|------|
| `nafxcwd.lib(appcore.obj) : error LNK2001 unresolved __argv/__argc` + LNK1120 | x64 配置误设为 UseOfMfc=Static，错链 32 位静态 MFC | x64 改为 `UseOfMfc=Dynamic` + runtime MultiThreadedDLL |
| `LNK2005: DllMain already defined` | 自定义了 DllMain，与 MFC 自带冲突 | 删除自己的 DllMain，锁改用静态全局对象 |
| `LNK4197: 多次指定导出` | .def 与 __declspec(dllexport) 同时导出 | 去掉所有 __declspec(dllexport)，导出只留 .def |
| `LNK4017: DESCRIPTION 不支持目标平台` | .def 写了 DESCRIPTION 行 | 删除 DESCRIPTION |
| `C2018: unknown character 0xef 0xbb 0xbf` | 旧编译器(VC6)不认识 UTF-8 BOM | 仅验证用临时 GBK 副本；正式文件保持 BOM |
| 中文注释乱码 | 文件无 BOM 且代码页不匹配 | 统一转 UTF-8 带 BOM；重新打开文件或删除 .sdf |
| COM10+ 打不开 | 未加 `\\.\` 前缀 | BuildPortName 统一加前缀 |
| 每条命令都要发两次才成功 | 一次 ReadFile 读不满应答 | 改循环 ReadFile 收满或超时 |

## Validate（交付检查清单）

- [ ] 目录含 .sln，且 sln/vcxproj 的 ProjectGuid 一致
- [ ] 有 4 个工程配置（Debug/Release × Win32/x64）
- [ ] x64 配置为 UseOfMfc=Dynamic + /MD（禁用静态 MFC）
- [ ] 全部源码 UTF-8 带 BOM（字节验证 EF BB BF）
- [ ] 无自定义 DllMain；锁为静态全局对象
- [ ] .def EXPORTS 与 Interface 导出函数一一对应且无 DESCRIPTION
- [ ] dumpbin /EXPORTS 验证无修饰导出成功
- [ ] ctypes 实测试验成功（读/写命令各一条）
- [ ] ReadMe.txt 含 Python 调用示例

## 版本注意（未来工作）

- 若用户坚持"64 位 + 静态 MFC"：本机 VS2010 无法实现（x64 无静态 MFC 库），需 VS2013+；应先向用户说明并征得同意，而不是静默改成动态。
- 若未来升级 VS2013/2017+，x64 可用静态 MFC，则 x64 配置改回 `UseOfMfc=Static` + /MT，发布不再依赖 VC 运行库。