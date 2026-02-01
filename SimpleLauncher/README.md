# SimpleLauncher

跨平台外部进程启动插件，支持蓝图调用。

## 功能特性

- 预设配置管理（运行时可动态增删改）
- 支持同一预设启动多个实例
- 进程退出回调通知
- 游戏退出时自动停止非分离进程
- 跨平台支持（Windows / Linux / macOS）

## 安装

1. 将 `SimpleLauncher` 文件夹放入项目 `Plugins/` 目录
2. 重新生成项目文件并编译

## 配置

打开 **项目设置 → Plugins → Simple Launcher**，在 `Launchers` 中添加预设配置：

| 字段 | 说明 | 默认值 |
|------|------|--------|
| Key (FName) | 预设名称，用于 API 调用 | - |
| CommandLine | 完整命令行（可执行文件 + 参数） | - |
| bLaunchHidden | 是否隐藏进程窗口 | false |
| bLaunchDetached | 分离运行（true=游戏退出后继续运行） | false |
| WorkingDirectory | 工作目录（可选） | - |

### 配置示例

| 预设名 | CommandLine | bLaunchDetached |
|--------|-------------|-----------------|
| Server | `server.exe --port 8080` | false |
| Logger | `python logger.py` | true |

## API 参考

### 类型定义

#### FProcessId

进程标识结构体。

| 字段 | 类型 | 说明 |
|------|------|------|
| Name | FName | 预设名称 |
| Index | int32 | 进程索引（同一预设可启动多个实例） |

```cpp
// 检查是否有效
bool IsValid() const;
```

#### FLauncherConfig

预设配置结构体。

| 字段 | 类型 | 说明 |
|------|------|------|
| CommandLine | FString | 完整命令行 |
| bLaunchHidden | bool | 隐藏窗口 |
| bLaunchDetached | bool | 分离运行 |
| WorkingDirectory | FString | 工作目录 |

### 配置管理

#### UpdateConfig

更新或创建预设配置。

```cpp
static bool UpdateConfig(
    const FName PresetName,
    const FLauncherConfig& Config,
    bool bOverrideIfExists = true,
    bool bCreateIfNotExists = true
);
```

| 参数 | 说明 |
|------|------|
| PresetName | 预设名称 |
| Config | 配置内容 |
| bOverrideIfExists | 存在时是否覆盖 |
| bCreateIfNotExists | 不存在时是否创建 |
| 返回值 | 是否成功 |

#### GetConfig

获取预设配置。

```cpp
static bool GetConfig(const FName PresetName, FLauncherConfig& OutConfig);
```

#### GetAllConfig

获取所有预设配置。

```cpp
static TMap<FName, FLauncherConfig> GetAllConfig();
```

#### RemoveConfig

删除预设配置（同时停止该预设的所有进程）。

```cpp
static bool RemoveConfig(const FName PresetName);
```

### 进程启动

#### StartProcess

启动预设进程。

```cpp
static FProcessId StartProcess(const FName PresetName);
```

| 参数 | 说明 |
|------|------|
| PresetName | 预设名称 |
| 返回值 | 进程标识（无效表示失败） |

### 进程停止

#### StopProcess

停止单个进程。

```cpp
static bool StopProcess(const FProcessId& ProcessId);
```

#### StopProcesses

停止预设的所有进程。

```cpp
static void StopProcesses(const FName PresetName);
```

#### StopAllProcesses

停止所有进程。

```cpp
static void StopAllProcesses();
```

### 进程查询

#### IsProcessRunning

检查单个进程是否运行中。

```cpp
static bool IsProcessRunning(const FProcessId& ProcessId);
```

#### HasRunningProcess

检查预设是否有运行中的进程。

```cpp
static bool HasRunningProcess(const FName PresetName);
```

#### GetProcessCount

获取预设的运行进程数量。

```cpp
static int32 GetProcessCount(const FName PresetName);
```

#### GetProcesses

获取预设的所有进程标识。

```cpp
static TArray<FProcessId> GetProcesses(const FName PresetName);
```

### 回调管理

#### BindProcessExited

绑定单个进程的退出回调。

```cpp
static void BindProcessExited(const FProcessId& ProcessId, const FOnProcessExitedDelegate& Delegate);
```

#### UnbindProcessExited

解绑单个进程的退出回调。

```cpp
static void UnbindProcessExited(const FProcessId& ProcessId, const FOnProcessExitedDelegate& Delegate);
```

#### BindPresetExited

绑定预设全部进程的退出回调。

```cpp
static void BindPresetExited(const FName PresetName, const FOnProcessExitedDelegate& Delegate);
```

#### UnbindPresetExited

解绑预设全部进程的退出回调。

```cpp
static void UnbindPresetExited(const FName PresetName, const FOnProcessExitedDelegate& Delegate);
```

### 回调触发行为

| 场景 | 触发回调 | bNaturalExit |
|------|----------|--------------|
| 进程自然退出 | ✅ | true |
| StopProcess | ✅ | false |
| StopProcesses | ✅ | false |
| RemoveConfig | ✅ | false |
| 游戏退出自动停止 | ❌ | - |

## 使用示例

### 蓝图

1. 在项目设置中添加预设 `MyServer`
2. 调用 `Start Process` 节点，PresetName 填入 `MyServer`
3. 返回的 `FProcessId` 可用于后续操作（停止、查询、绑定回调）

### C++

```cpp
#include "SimpleLauncherGameSubsystem.h"

// 启动进程
FProcessId ProcessId = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));

// 检查是否有效
if (ProcessId.IsValid())
{
    // 绑定退出回调
    // ...
}

// 启动多个实例
FProcessId Instance1 = USimpleLauncherGameSubsystem::StartProcess(FName("Worker"));
FProcessId Instance2 = USimpleLauncherGameSubsystem::StartProcess(FName("Worker"));

// 停止单个
USimpleLauncherGameSubsystem::StopProcess(Instance1);

// 停止预设的所有进程
USimpleLauncherGameSubsystem::StopProcesses(FName("Worker"));

// 运行时创建预设
FLauncherConfig Config;
Config.CommandLine = TEXT("python script.py");
Config.bLaunchHidden = true;
USimpleLauncherGameSubsystem::UpdateConfig(FName("TempTask"), Config);
USimpleLauncherGameSubsystem::StartProcess(FName("TempTask"));
```

## 命令行格式

| 格式 | 示例 |
|------|------|
| 无空格路径 | `notepad.exe file.txt` |
| 双引号包裹 | `"C:\Program Files\app.exe" -arg` |
| 单引号包裹 | `'/usr/bin/app' -arg` |

## 平台支持

| 平台 | 可执行类型 |
|------|------------|
| Windows | `.exe` `.bat` `.cmd` |
| Linux | 二进制文件、`.sh` 脚本 |
| macOS | 二进制文件、`.sh` 脚本 |

## 分离运行说明

`bLaunchDetached` 控制进程与游戏的关联关系：

| bLaunchDetached | 说明 | 游戏正常退出 | 游戏崩溃 |
|-----------------|------|--------------|----------|
| false（默认） | 关联运行 | 自动停止进程 | 系统可能终止进程 |
| true | 分离运行 | 进程继续运行 | 进程继续运行 |

**使用场景**：
- `false`：游戏服务器、临时工具（游戏结束后不需要）
- `true`：日志收集器、后台服务（游戏结束后仍需运行）

## 注意事项

- 删除预设（`RemoveConfig`）会停止所有进程并清理相关回调
- 更新预设（`UpdateConfig`）不会影响已运行的进程，新配置下次启动生效
- 退出回调的 `bNaturalExit` 参数：`true` 表示进程自然退出，`false` 表示被 API 关闭
- 游戏退出时自动停止的进程不触发回调

## 版本要求

- Unreal Engine 5.2+

## 联系方式

如有问题或反馈，请在 Fab 产品页面留言。
