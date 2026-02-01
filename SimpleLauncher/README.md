# SimpleLauncher

跨平台外部进程启动插件，支持蓝图调用。

## 功能特性

- 通过配置名快速启动预设进程
- 支持运行时启用/禁用配置
- 跨平台支持（Windows / Linux / macOS）
- 蓝图完整支持

## 安装

1. 将 `SimpleLauncher` 文件夹放入项目 `Plugins/` 目录
2. 重新生成项目文件并编译

## 配置

打开 **项目设置 → Plugins → Simple Launcher**，在 `Launchers` 中添加配置：

| 字段 | 说明 |
|------|------|
| Key (FName) | 配置名称，用于 API 调用 |
| bEnabled | 是否启用 |
| CommandLine | 完整命令行（可执行文件 + 参数） |
| bLaunchHidden | 是否隐藏进程窗口 |
| WorkingDirectory | 工作目录（可选） |

### 配置示例

| 配置名 | CommandLine | 说明 |
|--------|-------------|------|
| Notepad | `notepad.exe` | 打开记事本 |
| OpenFile | `notepad.exe C:\test.txt` | 打开指定文件 |
| RunScript | `python script.py --verbose` | 运行 Python 脚本 |
| BatchJob | `cmd /c build.bat arg1 arg2` | 执行批处理 |

## 蓝图 API

### 配置启动

| 函数 | 说明 |
|------|------|
| `Launch(Name)` | 通过配置名启动进程，返回是否成功 |
| `SetLauncherEnabled(Name, bEnabled)` | 设置配置启用状态 |
| `IsLauncherEnabled(Name)` | 检查配置是否启用 |

### 直接启动

| 函数 | 说明 |
|------|------|
| `LaunchWithConfig(Config)` | 传入配置结构启动 |
| `LaunchProcess(CommandLine)` | 传入命令行字符串启动 |

## 平台支持

| 平台 | 可执行类型 |
|------|------------|
| Windows | `.exe` `.bat` `.cmd` `.ps1`（需 powershell 调用） |
| Linux | 二进制文件、`.sh` 脚本 |
| macOS | 二进制文件、`.sh` `.command` 脚本 |

## 使用示例

### 蓝图

1. 在项目设置中添加配置 `MyApp`，CommandLine 设为 `notepad.exe`
2. 蓝图中调用 `Launch` 节点，Name 填入 `MyApp`

### C++

```cpp
#include "SimpleLauncherLibrary.h"

// 通过配置名启动
USimpleLauncherLibrary::Launch(FName("MyApp"));

// 直接命令行启动
USimpleLauncherLibrary::LaunchProcess(TEXT("notepad.exe C:\\test.txt"));

// 运行时禁用配置
USimpleLauncherLibrary::SetLauncherEnabled(FName("MyApp"), false);
```

## 版本要求

- Unreal Engine 5.2+

## 联系方式

如有问题或反馈，请在 Fab 产品页面留言。
