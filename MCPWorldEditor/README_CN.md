[English](./README.md) | [中文](./README_CN.md)

# MCPWorldEditor

基于 MCP（Model Context Protocol）的 Unreal Engine AI 世界与关卡编辑插件。

## 概述

MCPWorldEditor 在 Unreal Editor 内运行 HTTP 服务，将 Actor 创建、变换与属性编辑、灯光、地形、关卡和视口操作暴露给 AI 助手。

> **注意**：插件仍在开发中；服务器基础设施已经可用，工具能力会持续完善。

## 配置

在 `Project Settings > Plugins > MCP World Editor` 中配置：

| 设置 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| Port | int32 | 8764 | HTTP 端口；占用时最多自动递增重试 10 次 |
| bAutoStart | bool | true | 编辑器启动时自动启动服务 |
| RequestTimeoutSeconds | int32 | 30 | 工具执行超时时间 |
| MaxRequestBodyBytes | int32 | 1048576 | UTF-8/JSON 解析前允许的最大 HTTP 请求体 |
| MaxResponseBodyBytes | int32 | 8388608 | 允许返回的最大 UTF-8 JSON 响应；截图 Base64 也受此预算约束 |
| MaxCapturePixels | int32 | 8294400 | `CaptureViewport` 允许读取的最大像素数（默认约为 4K） |

`CaptureViewport` 在 PNG 编码前会将视口 Alpha 统一为完全不透明，确保保存文件和内嵌截图可在标准图片查看器及 Fab 工作流中正确显示。

## MCP 连接

服务地址为 `http://127.0.0.1:8764/mcp`。若默认端口被占用，请以编辑器工具栏显示的实际地址为准。

### Claude Code

在全局 `~/.claude/.mcp.json` 或项目根目录 `.mcp.json` 中加入：

```json
{
  "mcpServers": {
    "ue-world-editor": {
      "url": "http://127.0.0.1:8764/mcp"
    }
  }
}
```

### 其他 AI 客户端

Cursor、Windsurf、VS Code Copilot 等客户端的配置位置不同，请按对应客户端文档添加同一 MCP 服务地址。

工具请求在进入 Game Thread 前超时会被取消，可在安全确认后重试。若请求已被 Game Thread 领取，错误内容会返回 `code=timeout_execution_in_progress`、`retryable=false` 与 `stateMayHaveChanged=true`；此时操作仍可能完成，调用方必须先查询编辑器状态，不能盲目重试。

响应大小在工具执行完成后核验。若 `tools/call` 的结果超出预算，JSON-RPC 错误数据会返回 `code=response_too_large`、`retryable=false` 与 `stateMayHaveChanged=true`；这不表示工具被回滚，调用方同样必须先核验编辑器状态。

服务器会在请求进入 Game Thread 前统一按每个工具公布的 `inputSchema` 校验顶层必填字段、精确 JSON 类型、数组元素类型、数组 `maxItems` 和未知字段；所有工具 schema 默认 `additionalProperties=false`，现有批量数组统一公布 100 项上限。业务语义（资产路径、数值范围、对象存在性等）仍由具体工具执行第二层校验。

PIE 启动请求排队后即视为编辑器忙碌；`PIEControl/GetState` 与 `GetEditorState` 会区分 `Idle`、`StartQueued` 和 `Playing`，`Stop` 可取消尚未开始的排队请求。视口类型与渲染模式工具只使用已知的关卡编辑器视口，不会把资产编辑器预览视口强制下转；同时提供 ViewMode 与 ShowFlag 时会先完成两者校验再应用，错误不会留下部分渲染模式变更。

`ListActors` 与 `GetSelectedActors` 返回 `ActorPath`、`FName`、`Label` 和 `LevelPath`。所有通过 `ActorName/ActorNames` 寻址现有 Actor 的工具都优先接受完整 `ActorPath`；旧的 Label/FName 只有在当前 World 中唯一命中时才会执行，多命中会拒绝操作并返回排序后的候选路径，避免对错误 Actor 进行选择、移动、属性修改或删除。Actor 被重命名或移入其他关卡后路径会变化，调用方应重新查询。

## 要求

- Unreal Engine 5.2+
- 仅支持编辑器

## 工具与编辑器状态合同

当前源码版本注册了 71 个工具，且 71 个工具名称均唯一。

`RemoveStreamingLevel` 操作本身不可撤销。它会请求 Unreal Engine 保留既有 Undo 缓冲，但引擎仍可能因过期引用而重置该缓冲，因此不能保证 Undo 历史一定保留。工具只移除属于目标关卡的 Actor、Component、Object 与 BSP Selection；它会快照并核验目标关卡之外已选 Actor、Component 与 Object 的恢复，其他关卡中的 BSP Selection 则保持原状而不会被全局清除。若移除失败，工具还会尝试恢复目标关卡内的选择，并报告恢复是否通过核验。

`MoveActorsToLevel` 会尝试恢复并核验调用前的 Actor、Component、Object 与 BSP Selection；无法确认完整恢复时会返回错误。Selection Lock 启用或 Actor 身份无法唯一映射时，工具会在移动前拒绝请求。若批次结果不一致，工具不会自动执行 Undo，因为无法证明栈顶事务属于本次调用；继续操作前必须同时检查 Actor 所在关卡与 Selection 状态。

## World Partition 区域加载

`LoadRegion` 通过 Unreal Engine 公开的 World Partition 加载器 API 创建并加载持久的编辑器区域。`BoundsMin` 与 `BoundsMax` 都必须包含有限的数值字段 `X`、`Y`、`Z`，且每个轴的最小值必须严格小于对应最大值。无效边界会在创建加载器之前被拒绝。

## 材质实例创建

`CreateMaterialInstance` 只接受项目 `/Game/` 内容根目录下可写的 long package name，例如 `/Game/Materials/MI_Wall`。工具会拒绝 Developers 与自动生成的 ExternalActors/ExternalObjects 目录、非法包名，以及内存或磁盘上已经存在的目标。父级可以是任何可加载的 `MaterialInterface`，包括材质实例。

## Actor 编组

`GroupActors` 会在执行变更前解析并校验完整 Actor 列表、拒绝重复目标与跨关卡编组，并直接调用 Unreal Engine 接受 Actor 数组的编组 API。工具不会为了执行操作而清空或替换编辑器 Selection，因此会保留仍然存活的 Actor 的既有选择；若一个已选中的 GroupActor 被明确取消编组并销毁，该已移除对象会自然从 Selection 中消失。

## Actor 附加事务

`AttachActor` 与 `DetachActor` 会把 Actor 及实际参与父子关系变更的场景组件一并纳入编辑器事务，因此 Undo/Redo 能恢复组件层级、socket 与变换。若引擎拒绝操作或操作后的父级状态不符合请求，工具会尝试显式恢复原父组件、socket 与变换后返回错误；错误中的 `Original attachment restored` 表明恢复是否已通过验证，值为 `false` 时必须将编辑器状态视为可能已变化并先行检查。

## 组件创建安全

`AddComponent` 接受精确的完整组件类路径，或在当前已加载类中唯一匹配的短类名；短名存在多个候选时会返回排序后的完整路径并拒绝创建。工具会拒绝非 `ActorComponent`、抽象、弃用、已被新版本替代以及 `ClassWithin` 与目标 Actor 不兼容的类。实例组件按 Unreal Editor 的创建顺序加入序列化数组、通知创建并注册；附加、设为根组件或注册失败时会清理新组件并取消事务。

## World Settings 编辑

`GetWorldSettings` 返回当前实例上所有此刻可编辑属性的精确名称与 UE 文本值，不再只显示偏离默认值的属性，也不会把只读的 BlueprintVisible 字段当作可写入口。`SetWorldSettings` 要求大小写完全匹配，拒绝编辑标志或 `CanEditChange` 禁止的属性，并以成对的 `PreEditChange`/`PostEditChangeProperty` 通知完成事务。导入失败时会先恢复旧值；若恢复也失败，错误会报告事务回滚结果，回滚失败表示编辑器状态可能已变化。

## World Partition 区域生命周期

`LoadRegion` 只接受有限且逐轴递增的边界；单轴最长 1,000,000 Unreal 单位、总体积最多 1e17 立方单位，同时最多保留 16 个由 MCP 创建的活动区域。成功结果返回不可推测的 `RegionHandle`，调用 `UnloadRegion` 可卸载并释放对应的官方 Editor Loader Adapter。句柄不会按 bounds 猜测或复用；切换/清理 World 以及卸载插件模块时，插件会自动释放仍由它管理的区域，此后旧句柄会失效。

## 受限控制台命令

`ExecuteConsoleCommand` 只允许文档列出的七条 `stat` 命令，通过 `GEngine->Exec` 进入引擎命令链，并检查命令是否真正被处理；未处理时返回错误。UE 的 `stat` 命令切换对应的视口统计显示，重复调用可能将显示关闭；它们不是无副作用的只读测量。成功结果中的 `Handled=true` 只表示命令已由当前引擎实例处理，不表示统计显示一定处于开启状态；部分命令不会产生文本输出。

## Material Instance 资产持久化

`CreateMaterialInstance` 只允许在项目 `/Game/...` 根下、且不属于 Developers 或生成的 ExternalActors/ExternalObjects 目录的合法新包路径。工具会同时拒绝内存或磁盘上已存在的包，创建后通过官方 `EditorAssetSubsystem` 保存，并核验实际对象路径、磁盘包可发现且 package 不再为 dirty；只有全部成立才返回 `Saved=true`。保存或类型/路径校验失败时，只会在返回对象精确匹配本次目标时强制清理；只有目标对象、内存包和磁盘包均不再存在才报告清理成功。异常对象不匹配目标时拒绝盲删，并明确提示可能残留部分状态。资产创建本身不属于普通 Undo 事务；但失败清理使用 UE Force Delete，可能清空编辑器已有的 Undo 历史。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
