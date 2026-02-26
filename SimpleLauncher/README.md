# SimpleLauncher 使用指南

## 快速上手

只需两步即可从游戏中启动外部程序：

**第 1 步：配置预设**

打开 **项目设置 → Plugins → Simple Launcher**，在 Launchers 中添加一条预设：
- Key 填预设名称，如 `MyServer`
- CommandLine 填要执行的命令，如 `notepad.exe` 或 `python script.py`

**第 2 步：调用 API 启动**

在蓝图或 C++ 中调用：
```cpp
FProcessId Id = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
```

程序就会启动。游戏退出时，插件会自动帮你关闭它。

## 配置项说明

每个预设包含以下配置：

### Key（预设名称）

给这个预设起个名字，后续所有 API 都通过这个名字来操作。

- 可以随便取，如 `MyServer`、`Logger`、`DataSync`
- 名字不能重复，每个预设的名字是唯一的

### CommandLine（命令行）

要执行的完整命令，包括程序路径和参数。

```
notepad.exe file.txt
"C:\Program Files\app.exe" -port 8080
'/usr/bin/app' -arg
```

> 路径包含空格时，必须用引号包裹程序路径，否则会解析失败。

### bLaunchHidden（隐藏窗口）

是否隐藏程序窗口。默认 `false`（显示窗口）。

- 设为 `true`：程序在后台运行，不弹出窗口
- 适合后台服务、不需要用户看到的工具

### bLaunchDetached（分离运行）

控制游戏退出时程序是否跟着关闭。默认 `false`（跟着关闭）。

- `false`（默认）：游戏退出时自动关闭该程序 —— 适合游戏服务器、临时工具
- `true`：游戏退出后程序继续运行 —— 适合日志收集器等需要持续运行的服务

### WorkingDirectory（工作目录）

程序的工作目录。默认为空（使用游戏的工作目录）。

- 填绝对路径，如 `C:\MyApp` 或 `/home/user/app`
- 程序中使用的相对路径会基于这个目录解析

## API 说明

所有 API 都是 `USimpleLauncherGameSubsystem` 的静态函数，蓝图和 C++ 均可调用。

### 配置管理

| 函数 | 说明 |
|------|------|
| `UpdateConfig(Name, Config, bOverrideIfExists, bCreateIfNotExists)` | 创建或更新预设 |
| `GetConfig(Name, OutConfig)` | 获取预设配置 |
| `GetAllConfig()` | 获取所有预设 |
| `RemoveConfig(Name)` | 删除预设（同时停止该预设的所有进程） |

`UpdateConfig` 有两个可选参数，用于精确控制行为：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `bOverrideIfExists` | `true` | 预设已存在时是否覆盖，设为 `false` 可防止意外覆盖 |
| `bCreateIfNotExists` | `true` | 预设不存在时是否创建，设为 `false` 则只更新已有预设 |

除了在项目设置中配置，也可以在运行时用代码动态创建预设：

```cpp
FLauncherConfig Config;
Config.CommandLine = TEXT("python script.py");
Config.bLaunchHidden = true;
USimpleLauncherGameSubsystem::UpdateConfig(FName("TempTask"), Config);
```

> 运行时创建的预设在游戏重启后会丢失。如果需要持久保存，请在项目设置中配置。

### 进程控制

| 函数 | 说明 |
|------|------|
| `StartProcess(Name)` | 启动进程，返回 ProcessId |
| `StopProcess(ProcessId)` | 停止单个进程 |
| `StopProcesses(Name)` | 停止该预设的所有进程 |
| `StopAllProcesses()` | 停止全部进程 |

同一个预设可以多次调用 `StartProcess`，每次都会启动一个新的进程，返回不同的 `ProcessId`：

```cpp
FProcessId Id1 = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
FProcessId Id2 = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
// Id1 和 Id2 是两个独立运行的进程
```

### 状态查询

| 函数 | 说明 |
|------|------|
| `IsProcessRunning(ProcessId)` | 检查某个进程是否还在运行 |
| `HasRunningProcesses(Name)` | 该预设是否有正在运行的进程 |
| `GetProcessCount(Name)` | 该预设有多少个正在运行的进程 |
| `GetProcesses(Name)` | 获取该预设所有正在运行的 ProcessId |

### 进程退出回调

当进程退出时，可以收到通知并执行自定义逻辑（比如重启、记日志）。

| 函数 | 说明 |
|------|------|
| `BindProcessExited(ProcessId, Delegate)` | 监听某个进程的退出 |
| `UnbindProcessExited(ProcessId, Delegate)` | 取消监听某个进程 |
| `BindProcessesExited(Name, Delegate)` | 监听该预设下所有进程的退出 |
| `UnbindProcessesExited(Name, Delegate)` | 取消监听该预设 |

回调函数会收到两个参数：
- `ProcessId` —— 哪个进程退出了
- `bNaturalExit` —— 退出原因：`true` 表示程序自己结束的，`false` 表示被代码主动关闭的

**什么时候会触发回调：**

| 场景 | 触发回调 | bNaturalExit |
|------|----------|--------------|
| 程序自己运行完毕或崩溃 | 触发 | true |
| 调用 StopProcess 关闭 | 触发 | false |
| 调用 StopProcesses 关闭 | 触发 | false |
| 调用 RemoveConfig 删除预设 | 触发 | false |
| 游戏退出时自动清理 | 不触发 | - |

> 重复绑定相同的委托会自动忽略，不会重复触发。
>
> 游戏退出时不触发回调，是为了避免访问已销毁的对象导致崩溃。

## 注意事项

- `UpdateConfig` 只更新配置数据，不会影响已经在运行的进程。新配置在下次 `StartProcess` 时才会生效。
- `RemoveConfig` 删除预设时，会先停止该预设的所有进程。但已绑定的回调不会被清理，重新创建同名预设时可以继续使用。

### 平台支持

插件支持 Windows、Linux、macOS。底层使用 UE 的 `FPlatformProcess::CreateProc`，操作系统能执行的程序都可以启动，没有文件类型限制。

## 联系方式

如有问题或反馈，请在 Fab 产品页面留言。
