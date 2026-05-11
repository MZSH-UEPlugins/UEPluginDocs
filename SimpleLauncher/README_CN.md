[English](./README.md) | [中文](./README_CN.md)

# SimpleLauncher 用户指南

## 快速开始

只需两步从游戏中启动外部程序：

**步骤 1：配置预设**

打开 **项目设置 → Plugins → Simple Launcher**，在 Launchers 中添加预设：
- 设置 Key 为预设名称，如 `MyServer`
- 设置 CommandLine 为要执行的命令，如 `notepad.exe` 或 `python script.py`

**步骤 2：调用 API 启动**

从蓝图或 C++ 调用：
```cpp
FProcessId Id = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
```

程序将启动。当游戏退出时，插件会自动关闭它。

## 配置

每个预设包含以下设置：

### Key（预设名称）

此预设的名称。所有 API 通过此名称引用预设。

- 使用任意名称，如 `MyServer`、`Logger`、`DataSync`
- 名称在所有预设中必须唯一

### CommandLine

要执行的完整命令，包括程序路径和参数。

```
notepad.exe file.txt
"C:\Program Files\app.exe" -port 8080
'/usr/bin/app' -arg
```

> 如果路径包含空格，必须用引号包裹程序路径，否则解析会失败。

### bLaunchHidden

是否隐藏程序窗口。默认为 `false`（窗口可见）。

- 设为 `true`：程序在后台运行，无可见窗口
- 适用于不需要用户交互的后台服务或工具

### bLaunchDetached

控制游戏退出时是否关闭程序。默认为 `false`（随游戏关闭）。

- `false`（默认）：游戏退出时自动关闭程序——适用于游戏服务器、临时工具
- `true`：游戏退出后程序继续运行——适用于日志收集器或其他持久服务

### WorkingDirectory

程序的工作目录。默认为空（使用游戏的工作目录）。

- 使用绝对路径，如 `C:\MyApp` 或 `/home/user/app`
- 程序使用的相对路径将基于此目录解析

## API 参考

所有 API 都是 `USimpleLauncherGameSubsystem` 上的静态函数，可从蓝图和 C++ 调用。

### 配置管理

| 函数 | 说明 |
|------|------|
| `UpdateConfig(Name, Config, bOverrideIfExists, bCreateIfNotExists)` | 创建或更新预设 |
| `GetConfig(Name, OutConfig)` | 获取预设配置 |
| `GetAllConfig()` | 获取所有预设 |
| `RemoveConfig(Name)` | 移除预设（同时停止其所有进程） |

`UpdateConfig` 有两个可选参数用于精确控制：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `bOverrideIfExists` | `true` | 预设已存在时是否覆盖。设为 `false` 防止意外覆盖 |
| `bCreateIfNotExists` | `true` | 预设不存在时是否创建。设为 `false` 仅更新现有预设 |

除了在项目设置中配置预设，也可以在运行时动态创建：

```cpp
FLauncherConfig Config;
Config.CommandLine = TEXT("python script.py");
Config.bLaunchHidden = true;
USimpleLauncherGameSubsystem::UpdateConfig(FName("TempTask"), Config);
```

> 运行时创建的预设在游戏重启后丢失。持久预设请在项目设置中配置。

### 进程控制

| 函数 | 说明 |
|------|------|
| `StartProcess(Name)` | 启动进程，返回 ProcessId |
| `StopProcess(ProcessId)` | 停止单个进程 |
| `StopProcesses(Name)` | 停止预设的所有进程 |
| `StopAllProcesses()` | 停止所有进程 |

可以对同一预设多次调用 `StartProcess`。每次调用启动新进程并返回不同的 `ProcessId`：

```cpp
FProcessId Id1 = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
FProcessId Id2 = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
// Id1 和 Id2 是两个独立运行的进程
```

### 状态查询

| 函数 | 说明 |
|------|------|
| `IsProcessRunning(ProcessId)` | 检查特定进程是否仍在运行 |
| `HasRunningProcesses(Name)` | 检查预设是否有运行中的进程 |
| `GetProcessCount(Name)` | 获取预设的运行进程数 |
| `GetProcesses(Name)` | 获取预设所有运行中的 ProcessId |

### 进程退出回调

进程退出时获得通知并执行自定义逻辑（如重启、日志记录）。

| 函数 | 说明 |
|------|------|
| `BindProcessExited(ProcessId, Delegate)` | 监听特定进程退出 |
| `UnbindProcessExited(ProcessId, Delegate)` | 停止监听特定进程 |
| `BindProcessesExited(Name, Delegate)` | 监听预设下所有进程退出 |
| `UnbindProcessesExited(Name, Delegate)` | 停止监听预设 |

回调接收两个参数：
- `ProcessId` — 哪个进程退出了
- `bNaturalExit` — 退出原因：`true` 表示程序自行终止，`false` 表示被代码停止

**回调触发时机：**

| 场景 | 触发回调 | bNaturalExit |
|------|---------|--------------|
| 程序自行完成或崩溃 | 是 | true |
| 通过 StopProcess 停止 | 是 | false |
| 通过 StopProcesses 停止 | 是 | false |
| 通过 RemoveConfig 停止 | 是 | false |
| 游戏退出时自动清理 | 否 | - |

> 多次绑定相同委托会被静默忽略——不会触发重复回调。
>
> 游戏退出时不触发回调，以避免访问已销毁的对象导致崩溃。

## 注意事项

- `UpdateConfig` 仅更新配置数据，不影响已运行的进程。新配置在下次 `StartProcess` 调用时生效。
- `RemoveConfig` 在移除前会停止预设的所有进程。但绑定的回调不会被清除——如果重新创建同名预设，它们仍然有效。

### 平台支持

插件支持 Windows、Linux 和 macOS。底层使用 UE 的 `FPlatformProcess::CreateProc`，因此操作系统能执行的任何程序都可以启动——没有文件类型限制。

## 联系

如有问题或反馈，请在 Fab 产品页面留言。
