# SimpleLauncher

跨平台外部进程启动插件，支持蓝图调用。

## 功能特性

- 通过配置名启动/停止进程
- 游戏启动时自动启动进程
- 游戏结束时自动停止进程
- 跨平台支持（Windows / Linux / macOS）

## 安装

1. 将 `SimpleLauncher` 文件夹放入项目 `Plugins/` 目录
2. 重新生成项目文件并编译

## 配置

打开 **项目设置 → Plugins → Simple Launcher**，在 `Launchers` 中添加配置：

| 字段 | 说明 |
|------|------|
| Key (FName) | 配置名称，用于 API 调用 |
| CommandLine | 完整命令行（可执行文件 + 参数） |
| bLaunchHidden | 是否隐藏进程窗口 |
| WorkingDirectory | 工作目录（可选） |
| bAutoStart | 游戏启动时自动启动 |
| bAutoStop | 游戏结束时自动停止 |

### 配置示例

| 配置名 | CommandLine | bAutoStart | bAutoStop |
|--------|-------------|------------|-----------|
| Server | `server.exe --port 8080` | true | true |
| Logger | `python logger.py` | true | false |

## 蓝图 API

| 函数 | 说明 |
|------|------|
| `StartProcess(Name)` | 启动进程，返回是否成功 |
| `StopProcess(Name)` | 停止进程，返回是否成功 |
| `IsProcessRunning(Name)` | 检查进程是否运行中 |
| `StopAllProcesses()` | 停止所有进程 |

## 自动启动/停止

通过 `bAutoStart` 和 `bAutoStop` 配置，进程会在游戏启动/结束时自动管理：

- **bAutoStart = true**：游戏启动时自动启动该进程
- **bAutoStop = true**：游戏结束时自动停止该进程

## 平台支持

| 平台 | 可执行类型 |
|------|------------|
| Windows | `.exe` `.bat` `.cmd` |
| Linux | 二进制文件、`.sh` 脚本 |
| macOS | 二进制文件、`.sh` 脚本 |

## 使用示例

### 蓝图

1. 在项目设置中添加配置 `MyServer`
2. 蓝图中调用 `StartProcess` 节点，Name 填入 `MyServer`
3. 需要停止时调用 `StopProcess` 节点

### C++

```cpp
#include "SimpleLauncherLibrary.h"

// 启动进程
USimpleLauncherLibrary::StartProcess(FName("MyServer"));

// 检查是否运行
if (USimpleLauncherLibrary::IsProcessRunning(FName("MyServer")))
{
    // 停止进程
    USimpleLauncherLibrary::StopProcess(FName("MyServer"));
}

// 停止所有进程
USimpleLauncherLibrary::StopAllProcesses();
```

## 版本要求

- Unreal Engine 5.2+

## 联系方式

如有问题或反馈，请在 Fab 产品页面留言。
