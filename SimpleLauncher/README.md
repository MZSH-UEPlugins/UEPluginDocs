# SimpleLauncher User Guide

## Quick Start

Launch an external program from your game in just two steps:

**Step 1: Configure a Preset**

Open **Project Settings → Plugins → Simple Launcher**, and add a preset in Launchers:
- Set Key to a preset name, e.g. `MyServer`
- Set CommandLine to the command to execute, e.g. `notepad.exe` or `python script.py`

**Step 2: Call the API to Launch**

Call from Blueprint or C++:
```cpp
FProcessId Id = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
```

The program will start. When the game exits, the plugin will automatically shut it down.

## Configuration

Each preset contains the following settings:

### Key (Preset Name)

A name for this preset. All APIs reference presets by this name.

- Use any name you like, e.g. `MyServer`, `Logger`, `DataSync`
- Names must be unique across all presets

### CommandLine

The full command to execute, including the program path and arguments.

```
notepad.exe file.txt
"C:\Program Files\app.exe" -port 8080
'/usr/bin/app' -arg
```

> If the path contains spaces, you must wrap the program path in quotes, otherwise parsing will fail.

### bLaunchHidden

Whether to hide the program window. Defaults to `false` (window visible).

- Set to `true`: the program runs in the background with no visible window
- Ideal for background services or tools that don't need user interaction

### bLaunchDetached

Controls whether the program is shut down when the game exits. Defaults to `false` (shut down with game).

- `false` (default): the program is automatically closed when the game exits — ideal for game servers, temporary tools
- `true`: the program continues running after the game exits — ideal for log collectors or other persistent services

### WorkingDirectory

The working directory for the program. Defaults to empty (uses the game's working directory).

- Use an absolute path, e.g. `C:\MyApp` or `/home/user/app`
- Relative paths used by the program will resolve based on this directory

## API Reference

All APIs are static functions on `USimpleLauncherGameSubsystem`, callable from both Blueprint and C++.

### Config Management

| Function | Description |
|----------|-------------|
| `UpdateConfig(Name, Config, bOverrideIfExists, bCreateIfNotExists)` | Create or update a preset |
| `GetConfig(Name, OutConfig)` | Get preset configuration |
| `GetAllConfig()` | Get all presets |
| `RemoveConfig(Name)` | Remove a preset (also stops all its processes) |

`UpdateConfig` has two optional parameters for precise control:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `bOverrideIfExists` | `true` | Whether to overwrite if the preset already exists. Set to `false` to prevent accidental overwrites |
| `bCreateIfNotExists` | `true` | Whether to create if the preset doesn't exist. Set to `false` to only update existing presets |

In addition to configuring presets in Project Settings, you can create them dynamically at runtime:

```cpp
FLauncherConfig Config;
Config.CommandLine = TEXT("python script.py");
Config.bLaunchHidden = true;
USimpleLauncherGameSubsystem::UpdateConfig(FName("TempTask"), Config);
```

> Presets created at runtime are lost when the game restarts. For persistent presets, configure them in Project Settings.

### Process Control

| Function | Description |
|----------|-------------|
| `StartProcess(Name)` | Start a process, returns ProcessId |
| `StopProcess(ProcessId)` | Stop a single process |
| `StopProcesses(Name)` | Stop all processes for a preset |
| `StopAllProcesses()` | Stop all processes |

You can call `StartProcess` multiple times on the same preset. Each call launches a new process and returns a different `ProcessId`:

```cpp
FProcessId Id1 = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
FProcessId Id2 = USimpleLauncherGameSubsystem::StartProcess(FName("MyServer"));
// Id1 and Id2 are two independently running processes
```

### Status Queries

| Function | Description |
|----------|-------------|
| `IsProcessRunning(ProcessId)` | Check if a specific process is still running |
| `HasRunningProcesses(Name)` | Check if a preset has any running processes |
| `GetProcessCount(Name)` | Get the number of running processes for a preset |
| `GetProcesses(Name)` | Get all running ProcessIds for a preset |

### Process Exit Callbacks

Get notified when a process exits and execute custom logic (e.g. restart, logging).

| Function | Description |
|----------|-------------|
| `BindProcessExited(ProcessId, Delegate)` | Listen for a specific process exit |
| `UnbindProcessExited(ProcessId, Delegate)` | Stop listening for a specific process |
| `BindProcessesExited(Name, Delegate)` | Listen for all process exits under a preset |
| `UnbindProcessesExited(Name, Delegate)` | Stop listening for a preset |

The callback receives two parameters:
- `ProcessId` — which process exited
- `bNaturalExit` — exit reason: `true` means the program terminated on its own, `false` means it was stopped by code

**When callbacks are triggered:**

| Scenario | Callback Triggered | bNaturalExit |
|----------|-------------------|--------------|
| Program finishes or crashes on its own | Yes | true |
| Stopped via StopProcess | Yes | false |
| Stopped via StopProcesses | Yes | false |
| Stopped via RemoveConfig | Yes | false |
| Auto-cleanup on game exit | No | - |

> Binding the same delegate multiple times is silently ignored — it won't trigger duplicate callbacks.
>
> Callbacks are not triggered during game exit to avoid accessing destroyed objects, which would cause crashes.

## Notes

- `UpdateConfig` only updates configuration data and does not affect already running processes. The new configuration takes effect on the next `StartProcess` call.
- `RemoveConfig` stops all processes for the preset before removing it. However, bound callbacks are not cleared — they remain active if you recreate a preset with the same name.

### Platform Support

The plugin supports Windows, Linux, and macOS. It uses UE's `FPlatformProcess::CreateProc` under the hood, so any program the OS can execute can be launched — there are no file type restrictions.

## Contact

For questions or feedback, please leave a message on the Fab product page.
