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

### 保存、地图检查与构建

`GetDirtyPackages` 可按 `CurrentLevel` 或 `AllLoaded` 范围列出已加载脏包；`CurrentLevel` 包含当前 Level 已加载的外部对象包，以及当前关卡单独存在且为 dirty 的 Map BuiltData 包。`SavePackages` 需要精确 `PackageNames`，默认 `DryRun=true`；先预览所选包的 `PlanHash`，再以 `DryRun=false` 和同一哈希执行。空包保存可能删除现有文件，须显式设置 `AllowDeleteEmptyPackages`。启用源控提供器时，本地保存合同会拒绝实际写入；保存结果逐包报告，并列出仍为 dirty 的包。旧 `SaveLevel` 无参数调用继续保存当前 Level 及引擎纳入的外部对象包，结果同时报告真实 Level 包路径和残留脏包。磁盘保存不属于编辑器 Undo。

`MapCheck` 在当前地图运行引擎地图检查，返回错误、警告、带 token 的可见消息和 `Complete`。如果 Message Log 的界面过滤隐藏消息，或结果被 `MaxMessages` 截断，`Complete=false`，不能据此报告检查通过。检查会新增当前地图的日志页，保留历史页和用户界面过滤设置。

`BuildNavigation` 与 `BuildLighting` 默认只预览，实际执行须传回对应 `PlanHash` 并设置 `DryRun=false`。普通地图和 World Partition 地图的导航构建均调用引擎公开的同步 `UNavigationSystemV1::Build`，范围是已注册且已加载可见的编辑器关卡（`Scope=LoadedVisibleEditorWorld`）。已加载但隐藏的关卡及未加载关卡不在本次范围内；预览以 `ExcludedLoadedHiddenLevelPaths` 列出被排除的已加载隐藏关卡。改变关卡可见性会使预览哈希失效，关卡关联或可见性切换期间须等待状态稳定再预览或执行。导航构建保留编辑器 Undo，不隐式执行 `MapCheck` 或保存包。World Partition 构建可能清除已加载导航数据块；`bCompleteWorldPartitionBuild=false`、`bSaved=false` 不代表完成整个分区世界的离线构建。灯光构建使用 Preview 品质，返回 `OperationId`；用 `GetBuildStatus` 查询终态。只有所有启用的灯光场景均已应用结果，灯光构建才报告成功。两个构建都不报告虚构的进度百分比，也不会自行保存地图。地图切换或卸载会使仍在运行的操作 ID 失效。

`FindNavigationPath` 接受世界坐标 `Start` 和 `End`，在当前已加载的默认导航数据上同步查询路径，返回是否找到、是否为部分路径、长度和有界点列；它不会触发构建。`MaxPoints` 为 2–1000，超过上限时结果标记 `bTruncated=true`。

请通过 MCP `tools/list` 查询当前工具及其输入 schema；可用工具会随插件版本变化。

### 植被与实例化网格

先用 `ListFoliageTypes` 查找已加载的植被类型及精确路径。`AddFoliageInstances` 通过静态网格 `MeshPath` 和世界坐标 `Transforms` 添加实例；旧的 `Locations` 只指定位置，旋转和缩放为默认值。编辑已加载植被时，用 `FoliageInstances` 的 `Operation=Query`、精确 `IFAPath` 和 `FoliageTypePath` 查询，通过 `Offset`、`Limit`（最多 100）分页。`Update`、`Remove` 或 `Move` 指定 `Indices` 前，先按默认 `bDryRun=true` 预览，再用相同请求、`bDryRun=false`、`ExpectedCurrentSnapshotHash` 和 `ExpectedPlanHash` 执行。`Update` 的变换数必须与索引数一致。`Move` 还需要另一个已加载的 `TargetIFAPath` 和 `ExpectedTargetSnapshotHash`；World Partition 中源与目标植被分区都须已加载，位置也须属于所选分区。索引仅对对应快照有效。

`ScatterFoliage` 在 WorldStatic 地面上预览确定性的散布候选。提供已有植被类型的精确 IFA/类型路径、XY 矩形（`MinX`、`MinY`、`MaxX`、`MaxY`）、允许的 Z 范围、`Seed`、`MaxSlopeDegrees`、`MinSpacing`，以及 `Count`（最多 100）或 `Density`；`MaxSamples` 限制采样次数。查看预览后，带两个返回的哈希执行。非植被 ISM/HISM 组件使用 `InstancedMeshes`：通过精确 `ComponentPath` 分页 `Query`，或用世界坐标 `Transforms` 和快照索引预览并执行 `Add`、`Update`、`Remove`。植被组件应使用 `FoliageInstances`。

`ConvertInstancedMeshes` 可将简单 `StaticMeshActor` 转为分组的 ISM/HISM 实例（`Direction=ActorsToInstances`、精确 `ActorPaths`，可选 `bUseHISM`），或将非植被组件中的指定实例转回 Actor（`Direction=InstancesToActors`、精确 `ComponentPath` 和 `Indices`）。先预览，再用 `ExpectedCurrentSnapshotHash` 与 `ExpectedPlanHash` 执行。默认保留源。设置 `bDeleteSource=true` 时会检查可识别的已加载 Actor 引用、组件引用者及关卡脚本引用；发现外部硬引用即拒绝删除，即使已设置 `bAllowLossyConversion=true`。预览和结果会报告 `ReferenceCoverageComplete=false`、`ReferenceCoverage` 和 `UninspectableReferenceSources`：`GCObjectReferencer` 背后的原生 `FGCObject` 所有者无法完整检查，未加载分区及项目范围软引用也未扫描。原生硬引用不会迁移，删除源可能破坏其语义。删除 Actor 还须设置 `bAllowLossyConversion=true` 并使用新预览的双哈希。计划哈希包含引用策略及观测到的覆盖情况，但不能证明不透明原生引用集合未改变。转换先创建并核验目标，再删除源；无法保留 Actor 身份及所有 Actor 专有或实例自定义数据。

### Actor 摆放与样条

`PlaceActors` 用 `Mode=Ground`、`Surface`、`Grid`、`Bounds` 或 `Vertex` 与 `ActorNames`（最多 100 个精确路径或唯一名称）移动现有 Actor。Ground、Surface 向下射线探测，并将变换后包围盒底部落在命中面；`bAlignNormal` 可让 Surface 的局部 +Z 对齐法线。Grid 用 `GridSize` 对世界坐标枢轴取整。Bounds 以 `ReferenceActor` 为参照，按 `Axis`、`Side`、`Gap` 依次放置。Vertex 在 `SearchRadius` 和 `MaxVertices` 范围内，寻找 `ReferenceActor` 的指定静态网格 `ReferenceComponent`、`LOD` 上最近的顶点。`Mode=Array` 则复制 `SourceActor` `Count` 次（1–100），用 `Spacing`、各轴扰动和 `Seed` 控制世界变换。保存前查看返回的最终变换。

普通样条组件先用 `GetSplineData` 的 `ActorName`、精确 `ComponentName` 和可选 `CoordinateSpace=Local|World` 查询。把返回的 `SnapshotHash` 传给 `ApplySplinePatch`，通过有序 `Operations` 执行 `Add`、`Remove`、`SetLocation`、`SetTangents`、`SetRotation`、`SetScale`、`SetType` 或 `SetClosedLoop`。先预览，正式执行需 `bDryRun=false` 与预览的 `PlanHash`；插入和删除会改变同一批后续操作的索引。`SetSplineMesh` 用局部坐标的 `Start`、`StartTangent`、`End`、`EndTangent` 编辑一个样条网格组件，也可指定截面缩放、滚转、前向轴和静态网格。Landscape 样条使用下文的专用工具。

`ApplySplinePatch` 支持 Blueprint SCS 样条组件，构造脚本重跑后会重新核验当前组件。已经由构造脚本改写，或配置为将点交给构造脚本处理的样条会在写入前拒绝。重建或读回失败时，Undo 事务中可能保留变更；重试前先检查返回状态。

### World Partition Data Layer 与 Level Instance

在 World Partition 关卡中，先用 `GetWorldPartitionInfo` 查找 Data Layer，再用 `CreateDataLayer` 创建、`SetDataLayer` 添加或移除 Actor。UE 5.2 创建时需要已有的 `DataLayerAssetPath`；后续版本创建私有层。使用 `ActorNames` 批量调用 `SetDataLayer` 时默认预览，执行须传入新近的 `ExpectedHash`。`ManageDataLayer` 使用精确 `DataLayerName` 与 `Action=Query|Rename|SetParent|SetVisible|SetLoaded|SetInitialRuntimeState|Delete`。`SetParent` 用 `ParentName`（空字符串清除父层）；显示和加载状态用 `bValue`；初始运行状态可选 `Unloaded`、`Loaded`、`Activated`。删除先预览，再以返回的 `StateHash` 作为 `ExpectedHash`、设置 `bDryRun=false` 执行。资产支持的层不可独立重命名。

`ManageLevelInstance` 的 `Action=Query` 可列出实例。`Create` 需要 `ActorNames` 与新的 `/Game/...` `LevelPackagePath`；先预览，再用 `ExpectedHash` 执行。创建时先保存新目标关卡，但源关卡仍为 dirty：结果的 `SourceSaved=false` 与 `PendingSavePackages` 指明待保存包。移除源 Actor 前会检查已知已加载硬引用；未加载引用与软引用不扫描。阅读 `SoftReferenceRisk`，在预览与正式执行中设置 `bAcknowledgeUnscannedSoftReferences=true`，使用包含该确认的预览哈希，并另行保存报告的源包。创建失败可能留下目标地图或部分源变更，重试前先检查返回状态。

对精确 `InstanceActor` 使用 `EnterEdit`；返回的 `OperationId` 代表编辑会话，`MoveActors`、`Commit`、`Discard` 均须传入。`MoveActors` 先预览 Actor 批次，以新的 `ExpectedHash` 执行。`Break` 将实例展开为 Actor。`PackBlueprint` 在精确的 `/Game/...` `BlueprintPath` 创建或更新 Packed Level Actor Blueprint；`bPendingSave=true` 表示仍需保存 Blueprint。这些操作不代表源关卡或全部结果资产已保存。

Packed Blueprint 创建不弹出资产对话框，也不自动保存。在 UE 5.3 及更新版本中，引擎打包阶段会重置编辑器的全局 Undo 历史；如需撤销已有编辑，请在打包前完成。`PendingSavePackages` 列出包含已加载实例在内的潜在待保存包。创建失败可能保留新建的内存资产；错误信息会说明失败阶段和需要检查的包。

### Landscape 绘制层与专用样条

`GetLandscapeLayerInfo` 列出主 Landscape 的绘制层绑定。`CreateLandscapeLayerInfo` 用 `PackagePath`、`LayerName` 和 `bWeightBlended` 预览新的 `/Game/...` 资产，再以 `bDryRun=false`、`ExpectedPlanHash` 创建；资产还需单独保存。`BindLandscapeLayerInfo` 在已加载的非分区主 Landscape 上预览将已有 `LayerInfoPath` 绑定到 `LayerName`，或以 `bUnbind=true` 解绑。用新的 `ExpectedPlanHash` 执行；存在已绘制分配时拒绝解绑，且不会删除资产。

`GetLandscapeWeightData` 从精确绑定的 `LayerInfoPath` 读取端点包含在内的顶点区域（最多 65,536 个按行排列的 `uint8` 权重）；启用 Edit Layers 时指定 `EditLayerGuid`。`SetLandscapeWeightData` 接受精确 `WeightData`，或笔刷字段（`BrushCenterX/Y`、`BrushRadius`、`BrushTargetWeight`、`BrushStrength`）。它要求已加载、非分区的主 Landscape，且每个受影响组件已有目标分配。先预览有界区域，再用 `ExpectedCurrentDataHash` 和 `ExpectedProposedDataHash` 执行；权重混合层可能归一化其他绘制层的权重。`GetLandscapeEditLayers` 列出 Edit Layer GUID；`EditLandscapeLayer` 可预览创建、删除、重排、重命名、可见性、锁定、Alpha 或混合模式变更，执行须使用新的当前哈希和计划哈希。删除须 `bAllowDataLoss=true`；UE 5.8 不提供所需的公开编辑层混合模式 setter。

Landscape 专用样条先用 `GetLandscapeSplineTopology` 查看拓扑，再用 `PatchLandscapeSplineTopology` 执行 `AddPoint`、`UpdatePoint`、`DeletePoint`、`AddSegment`、`UpdateSegment` 或 `DeleteSegment`，按操作传入精确点/线段路径及局部坐标或切线。先预览，再以 `ExpectedTopologyHash` 与 `ExpectedPlanHash` 应用。该有界拓扑编辑要求已加载、非分区且没有保留样条 Edit Layer 的 Landscape。`ApplyLandscapeSplines` 可另外预览将样条作用于精确 `EditLayerGuid`，用 `ExpectedPlanHash` 应用；存在保留样条层时必须指定它的 GUID。目标层须可用且未锁定。这些操作不会保存关卡或资产。

### Landscape 高度数据安全合同

`GetLandscapeHeightData` 与 `SetLandscapeHeightData` 直接通过 JSON 交换按行排列的 `uint16` 高度采样，不探测或读取调用方指定的文件。区域使用包含端点的 Landscape 顶点坐标，单次最多 65,536 个采样。目标必须是主 `ALandscape` Actor，且整个区域必须已加载并归该 Actor 所有。UE 5.8 及更高版本要求精确指向标准编辑层的 `EditLayerGuid`，并拒绝程序化层与样条层；写工具还会拒绝锁定层，读工具则允许检查锁定层。UE 5.2-5.7 会拒绝包含 Edit Layers 的 Landscape，因为这些版本需要另行定义分层编辑合同。

`SetLandscapeHeightData` 默认使用 `bDryRun=true`。预检只返回 `CurrentDataHash` 与 `ProposedDataHash`，不改变编辑器状态。正式应用必须传入 `bDryRun=false`、刚刚预检得到的 `ExpectedCurrentDataHash`，以及匹配的 `ExpectedProposedDataHash`；当前状态或拟写入采样已变化时会在事务开始前拒绝。写入后工具会读回核验；核验失败时先恢复并验证原始采样，再取消事务，若恢复也无法确认则明确报告编辑器状态可能已变化。原子边界是单次区域调用，不是跨多次调用的整幅地形导入；两个工具均不会保存关卡或资产。

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
