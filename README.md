# mcu-serial-to-mfc-dll — opencode 技能

把单片机（STM32 等）串口文本协议封装成 Windows MFC 动态库（DLL）的方法（opencode skill）。

**触发方式**：说「转动态库」「生成DLL」「封装成库」「串口协议转DLL」，或要求「生成sln / UTF-8带BOM / 64位 / MFC库」。

## 流程概要

1. 分析固件串口协议（命令、应答长度、超时）
2. 创建 VS2010 工程（sln + vcxproj，Win32/x64 双平台）
3. 编写四层源码：串口底层 / 协议层 / 导出接口层 / MFC 入口
4. 统一转为 UTF-8 带 BOM 编码
5. 构建并用 dumpbin / Python ctypes 验证

## 硬性约定

| 项 | 约定 |
|----|------|
| VS 版本 | Visual Studio 2010 |
| 工程文件 | 必须含 .sln |
| 编码 | 全部源码 UTF-8 带 BOM |
| MFC 链接 | Win32 静态 MFC；x64 动态 MFC（VS2010 x64 无静态 MFC 库） |
| 导出 | 由 .def EXPORTS 控制，不写 __declspec(dllexport) |
| DllMain | 禁止自定义（MFC 自带，冲突） |

## 使用方式

放入 opencode 技能目录，对任一片机串口协议执行即可。

更多细节见 `SKILL.md`。