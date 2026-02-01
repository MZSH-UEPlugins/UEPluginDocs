# SimpleLauncher 使用指南

## 配置项说明

打开 **项目设置 → Plugins → Simple Launcher**，在 Launchers 中添加预设。

### Key（预设名称）

- **功能**：预设的唯一标识，用于 API 调用
- **填写**：任意名称，如 `MyServer`、`Logger`
- **示例**：`StartProcess(FName("MyServer"))`

### CommandLine（命令行）

- **功能**：要执行的完整命令
- **填写**：可执行文件路径 + 参数
- **格式**：
  - 无空格路径：`notepad.exe file.txt`
  - 有空格路径：`"C:\Program Files\app.exe" -arg`
  - Linux/Mac：`'/usr/bin/app' -arg`

### bLaunchHidden（隐藏窗口）

- **功能**：是否隐藏进程窗口
- **默认**：`false`（显示窗口）
- **设为 true**：进程在后台运行，无窗口
- **适用**：后台服务、不需要用户交互的工具

### bLaunchDetached（分离运行）

- **功能**：控制游戏退出时进程的行为
- **默认**：`false`（游戏退出时自动停止进程）
- **设为 true**：游戏退出后进程继续运行
- **适用**：
  - `false`：游戏服务器、临时工具
  - `true`：日志收集器、需要持续运行的后台服务

### WorkingDirectory（工作目录）

- **功能**：进程的工作目录
- **默认**：空（使用游戏的工作目录）
- **填写**：绝对路径，如 `C:\MyApp` 或 `/home/user/app`
- **影响**：进程中的相对路径会基于此目录解析

## API 说明

所有 API 通过 `USimpleLauncherGameSubsystem` 静态调用。

### 配置管理

| 函数 | 说明 |
|------|------|
| `UpdateConfig(Name, Config)` | 创建或更新预设 |
| `GetConfig(Name, OutConfig)` | 获取预设配置 |
| `GetAllConfig()` | 获取所有预设 |
| `RemoveConfig(Name)` | 删除预设并停止其进程 |

**运行时动态创建预设**：
```cpp
FLauncherConfig Config;
Config.CommandLine = TEXT("python script.py");
Config.bLaunchHidden = true;
USimpleLauncherGameSubsystem::UpdateConfig(FName("TempTask"), Config);
```

### 进程控制

| 函数 | 说明 |
|------|------|
| `StartProcess(Name)` | 启动进程，返回 ProcessId |
| `StopProcess(ProcessId)` | 停止单个进程 |
| `StopProcesses(Name)` | 停止预设的所有进程 |
| `StopAllProcesses()` | 停止全部进程 |

**启动进程**：
```cpp
FProcessId Id = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
```

**多实例**：同一预设可多次调用 StartProcess，每次返回不同的 ProcessId。

### 状态查询

| 函数 | 说明 |
|------|------|
| `IsProcessRunning(ProcessId)` | 检查单个进程是否运行中 |
| `HasRunningProcess(Name)` | 预设是否有运行中的进程 |
| `GetProcessCount(Name)` | 获取预设的运行进程数量 |
| `GetProcesses(Name)` | 获取预设的所有 ProcessId |

### 回调绑定

| 函数 | 说明 |
|------|------|
| `BindProcessExited(ProcessId, Delegate)` | 绑定单个进程的退出回调 |
| `UnbindProcessExited(ProcessId, Delegate)` | 解绑单个进程 |
| `BindPresetExited(Name, Delegate)` | 绑定预设所有进程的退出回调 |
| `UnbindPresetExited(Name, Delegate)` | 解绑预设 |

**回调参数**：
- `ProcessId` - 退出的进程标识
- `bNaturalExit` - `true` 进程自己结束，`false` 被 API 关闭

**回调触发规则**：

| 场景 | 触发回调 | bNaturalExit |
|------|----------|--------------|
| 进程自己结束 | ✅ | true |
| 调用 StopProcess | ✅ | false |
| 调用 RemoveConfig | ✅ | false |
| 游戏退出 | ❌ | - |

## 注意事项

### 命令行格式

路径包含空格时必须用引号包裹，否则会解析失败。

### 配置更新

- `UpdateConfig` 只更新配置，不影响已运行的进程
- 新配置在下次 `StartProcess` 时生效

### 删除预设

- `RemoveConfig` 会停止该预设的所有进程并清理回调
- 运行时创建的预设在游戏重启后丢失

### 游戏退出

- `bLaunchDetached = false` 的进程会被自动停止
- `bLaunchDetached = true` 的进程继续运行
- 游戏退出时不触发任何回调

### 平台支持

| 平台 | 支持的文件类型 |
|------|---------------|
| Windows | `.exe` `.bat` `.cmd` |
| Linux | 二进制文件、`.sh` 脚本 |
| macOS | 二进制文件、`.sh` 脚本 |

## 联系方式

如有问题或反馈，请在 Fab 产品页面留言。
