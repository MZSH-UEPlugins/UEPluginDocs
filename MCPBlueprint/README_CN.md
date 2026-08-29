[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint 用户教程

MCPBlueprint 通过 MCP 向 AI 客户端提供蓝图发现、图表编辑、成员、组件、资产生命周期、编译反馈和图表截图能力。当前编辑器会话中真实可用的工具名和参数始终以 `tools/list` 返回结果为准。

## 要求与安装

- Unreal Engine 5.2+ Editor，支持 Windows、macOS 和 Linux。
- 支持本机 HTTP 服务的 MCP 客户端。
- 插件包必须与 Unreal Engine 的小版本完全一致。

每个引擎只安装一次 MCPBlueprint：

```text
<UE_5.x>/Engine/Plugins/Marketplace/MCPBlueprint/
```

MCPBlueprint 是跟随引擎安装的可选编辑器工具。`.uplugin` 使用 `Installed=true` 和 `EnabledByDefault=true`。不要把它重复复制进每个项目，也不要向用户项目的 `.uproject` 添加 MCPBlueprint。替换引擎中的插件包后，应重启对应版本的编辑器。

## 连接 AI 客户端

MCPBlueprint 会自动启用，并在编辑器启动时自动启动本机 MCP 服务。用户不需要在每个项目中手动启用插件，也不需要向项目 `.uproject` 添加插件声明。

让你使用的 AI 客户端按照自身的 MCP 配置流程接入 MCPBlueprint，并把插件工具栏显示的端点交给它即可。默认端点为 `http://127.0.0.1:8766/mcp`；配置端口被占用时，MCPBlueprint 最多尝试后续九个端口。不同 AI 客户端的配置格式不同，因此本文不规定某个客户端专用的 JSON 文件或手动端口配置步骤。

`initialize` 和 `tools/list` 通常由 MCP 客户端自动完成；`ping` 只用于排障，不是普通用户每次都要手动执行的步骤。

服务器只绑定本机。浏览器风格的 `Origin` 只允许 `localhost`、`127.0.0.1` 和 `[::1]`；普通非浏览器 MCP 客户端可以省略 `Origin`。

## 当前已完成功能

当前版本注册 55 个工具：

| 能力 | 数量 | 工具 |
|---|---:|---|
| 发现与读取 | 6 | `ListBlueprints`、`GetBlueprintOverview`、`ListBlueprintMembers`、`GetGraphDetail`、`SearchGraphNodes`、`GetCompileErrors` |
| User Defined Struct 与 Enum | 6 | `CreateUserDefinedStruct`、`GetUserDefinedStruct`、`ModifyUserDefinedStruct`、`CreateUserDefinedEnum`、`GetUserDefinedEnum`、`ModifyUserDefinedEnum` |
| 变量、函数、事件与 Pin | 17 | `AddVariable`、`ModifyVariable`、`RemoveVariable`、`CreateFunction`、`RenameFunction`、`ModifyFunctionSignature`、`RemoveFunction`、`CreateCustomEvent`、`AddEventNode`、`AddBoundEvent`、`AddLocalVariable`、`RemoveLocalVariable`、`AddNodePin`、`RemoveNodePin`、`AddInterface`、`AddEventDispatcher`、`RemoveEventDispatcher` |
| 图表编辑与布局 | 4 | `ApplyGraphPatch`、`SetPinDefaults`、`FormatGraph`、`AddCommentBox` |
| 组件与默认值 | 11 | `GetComponents`、`AddComponent`、`SetComponentProperties`、`RenameComponent`、`RemoveComponent`、`ReparentComponent`、`SetRootComponent`、`DuplicateComponent`、`GetComponentProperties`、`GetClassDefaults`、`SetClassDefaults` |
| 资产生命周期与截图 | 11 | `CreateBlueprint`、`ReparentBlueprint`、`CompileBlueprint`、`SaveAsset`、`OpenAsset`、`CloseAsset`、`ReloadBlueprintFromDisk`、`DeleteAsset`、`RenameAsset`、`DuplicateAsset`、`CaptureGraphScreenshot` |

已完成的行为包括：

- 大型资产、成员、图表和 Pin 读取的稳定有界分页。
- 使用 Action Registry 返回的节点 ID，而不是编造节点类型。
- 单事务图表 Patch：创建、连接、断开、删除、移动、默认值和布局；失败时尝试自动回滚，恢复无法证明会明确报告。
- `Balanced`、`Straight`、`Compact` 三种确定性布局风格。
- 局部和全图布局中的原生 Comment Box 包围关系保护。
- 高风险修改的 dry-run 影响分析和显式批准门禁。
- 编译/读回验证、事务回滚，以及 Unreal 支持范围内的单步 Undo。
- 显式持久化：普通编辑工具只把资产标脏；`SaveAsset` 明确写盘，`DeleteAsset` 在更严格门禁后明确删除磁盘资产。
- 无需项目声明即可从引擎目录自动发现并加载。

## 概念区分：先选对编辑对象

下列对象在 UE 编辑器里常常相邻显示，但它们不是同一件事；选错工具可能得到“调用成功但效果不在预期位置”的结果。

| 要改的对象 | 含义与适用工具 | 不要混同为 |
|---|---|---|
| 变量默认值 | 本蓝图**声明的成员变量**的声明默认值；用 `ModifyVariable` 的 `DefaultValue`。 | 节点 Pin 的字面量，也不是关卡中已有实例的覆盖值。 |
| 函数参数 / Pin | 函数 Entry/Result 的参数是函数签名，须用 `ModifyFunctionSignature`；图中某个现有节点的输入 Pin 字面量用 `SetPinDefaults`。 | 成员变量默认值；已连接输入 Pin 的默认值不会参与运行。 |
| 组件模板属性 | 此蓝图 SCS 中添加的组件模板，例如 `RelativeLocation`、`StaticMesh`；用 `SetComponentProperties`。 | Class Defaults；也不是场景内 Actor 实例的组件覆盖值。 |
| Class Defaults | 该蓝图生成类的 CDO 默认属性，包含可编辑的继承属性；用 `SetClassDefaults`。 | `ModifyVariable` 的变量声明元数据，或已摆放/已生成实例的当前值。 |
| 蓝图父类 | Blueprint 类继承的父类；用 `ReparentBlueprint`。 | 组件树中的父组件关系。 |
| 组件父子级 | 同一个 Actor Blueprint 的 SceneComponent 附着层级；用 `ReparentComponent`，本地根组件改用 `SetRootComponent`。 | Blueprint 的类继承关系。 |

## 55 个工具逐项功能总表

表中“响应/读回证据”表示该工具没有对应的真实截图；请保留 MCP 响应，并按“如何验证”再次读取或编译。普通编辑工具只标脏，不会静默保存；`SaveAsset` 与 `DeleteAsset` 是明确的持久化例外。

| 工具 | 能做什么 | 关键输入 / 安全门禁 | 如何验证 | 截图 / 证据 |
|---|---|---|---|---|
| `ListBlueprints` | 分页列出蓝图资产 | `PathFilter`、分页参数 | 用返回路径打开或查概览 | 响应/读回证据 |
| `GetBlueprintOverview` | 读取父类、图、变量、组件、接口概览 | `BlueprintPath` | 与资产当前状态比对 | 响应/读回证据 |
| `ListBlueprintMembers` | 分页列函数、事件、分发器、局部变量 | `BlueprintPath`、分页参数 | 用成员名/图 GUID 再查详情 | 响应/读回证据 |
| `GetGraphDetail` | 读取图节点、GUID、Pin、连线与坐标 | `BlueprintPath`、可选 `GraphName`、分页参数 | 修改前后对比 GUID/Pin/连线 | 响应/读回证据 |
| `SearchGraphNodes` | 搜索合法 Action Registry `SpawnerId` | 查询文本、分页参数 | 仅使用响应中的 `SpawnerId` 创建节点 | 响应/读回证据 |
| `GetCompileErrors` | 编译蓝图并返回有界诊断 | `BlueprintPath` | 与返回的编译状态和诊断核对 | 响应/读回证据 |
| `CreateUserDefinedStruct` | 在 `/Game` 创建用户定义 Struct | `AssetPath`；创建后未自动保存 | `GetUserDefinedStruct` 后显式保存 | 响应/读回证据 |
| `GetUserDefinedStruct` | 读取 Struct 字段和稳定字段 GUID | `StructPath`、分页参数 | 保存前后读回 | 响应/读回证据 |
| `ModifyUserDefinedStruct` | 安全修改 Struct 字段 | `bDryRun`；可能数据损失须显式批准 | 读回 GUID/引用影响，编译相关蓝图 | 响应/读回证据 |
| `CreateUserDefinedEnum` | 在 `/Game` 创建用户定义 Enum | `AssetPath`；创建后未自动保存 | `GetUserDefinedEnum` 后显式保存 | 响应/读回证据 |
| `GetUserDefinedEnum` | 读取 Enum 条目、整数值与显示名 | `EnumPath`、分页参数 | 保存前后读回 | 响应/读回证据 |
| `ModifyUserDefinedEnum` | 安全修改 Enum 条目 | 先 `bDryRun`；序列化语义变化须批准 | 读回值、引用影响并编译 | 响应/读回证据 |
| `AddVariable` | 增加成员变量 | `BlueprintPath`、名称、`TypeName` | 概览/变量读回并编译 | 响应/读回证据 |
| `ModifyVariable` | 改成员变量类型、默认值、名、可编辑性、分类、提示 | `VariableName`；改类型有引用时拒绝，RepNotify 不能非交互改名 | 概览读回，编译，必要时保存 | 响应/读回证据 |
| `RemoveVariable` | 安全删除成员变量 | 受引用与恢复能力检查 | 概览、编译和受影响调用点读回 | 响应/读回证据 |
| `CreateFunction` | 创建蓝图函数 | `BlueprintPath`、函数名、参数定义 | `GetGraphDetail` 获取稳定 GUID | 响应/读回证据 |
| `RenameFunction` | 安全重命名用户函数并更新可安全更新的引用 | 先 dry-run；引用更新须显式批准 | 读回新图名/调用点，编译、保存、重开 | 见下方真实 UE5.2 截图 |
| `ModifyFunctionSignature` | 增加、改名、删除或改类型函数参数 | `FunctionGraphGuid`、稳定 `PinId`；默认 dry-run，写入/数据损失分别批准 | 读回签名与调用点，编译所有受影响蓝图 | 见下方当前运行时函数签名图；修改仍以读回为准 |
| `RemoveFunction` | 安全删除用户函数 | 引用/恢复检查 | 概览和调用点读回，编译 | 响应/读回证据 |
| `CreateCustomEvent` | 在 Event Graph 创建自定义事件 | 目标必须是真实 Event Graph | 图详情读回并编译 | 响应/读回证据 |
| `AddEventNode` | 添加受支持的事件节点 | 蓝图、事件类型 | 图详情读回并编译 | 响应/读回证据 |
| `AddBoundEvent` | 为组件/委托添加绑定事件 | 组件与委托必须可解析 | 图详情读回并编译 | 见下方当前运行时 Event Graph |
| `AddLocalVariable` | 在函数图添加局部变量 | `BlueprintPath`、`FunctionName`、变量名与类型 | `ListBlueprintMembers`/图详情读回 | 响应/读回证据 |
| `RemoveLocalVariable` | 删除函数局部变量 | 局部声明定位与引用安全检查 | 成员/图详情读回并编译 | 响应/读回证据 |
| `AddNodePin` | 为支持可变 Pin 的节点增加 Pin | `BlueprintPath`、`NodeGuid`、可选 `Count` | 图详情读回 | 响应/读回证据 |
| `RemoveNodePin` | 删除可编辑节点用户 Pin | 节点 GUID、Pin 定位 | 图详情读回并编译 | 响应/读回证据 |
| `AddInterface` | 为蓝图添加接口 | 接口类路径 | 概览接口列表及编译 | 响应/读回证据 |
| `AddEventDispatcher` | 添加事件分发器 | 蓝图、名称/签名 | 成员读回并编译 | 响应/读回证据 |
| `RemoveEventDispatcher` | 删除事件分发器 | 引用与安全检查 | 成员读回并编译 | 响应/读回证据 |
| `ApplyGraphPatch` | 单事务创建、连线、断线、删/移节点、设默认值与布局 | 先搜索 `SpawnerId`；支持 dry-run；失败时尝试自动回滚，无法证明恢复会明确报错 | `GetGraphDetail`、编译 | 响应/读回证据 |
| `SetPinDefaults` | 仅改现有节点输入 Pin 字面量 | `NodeGuid`、非空 `PinDefaults`；已连 Pin 保持连线并在运行时忽略字面量 | 图详情读回目标 Pin | 响应/读回证据 |
| `FormatGraph` | 以 `Balanced`/`Straight`/`Compact` 布局图 | 图定位、布局范围/风格 | 图详情坐标或截图读回 | 响应/读回证据 |
| `AddCommentBox` | 添加原生 Comment Box | 图、文字、范围/节点 | 图详情读回 | 响应/读回证据 |
| `GetComponents` | 读取组件树、父级和模板信息 | Actor Blueprint 路径 | 用精确组件名再查属性 | 响应/读回证据 |
| `AddComponent` | 添加本地 SCS 组件 | 组件类、名称、可选父组件 | `GetComponents` 后编译 | 响应/读回证据 |
| `SetComponentProperties` | 原子修改本蓝图 SCS 组件模板属性 | `ComponentName`、`Properties`（UE 文本格式）；失败整笔回滚 | `GetComponentProperties`、编译 | 待补真实截图 |
| `RenameComponent` | 重命名本地 SCS 组件并更新成员引用 | 当前名、新唯一名 | `GetComponents`、编译 | 响应/读回证据 |
| `RemoveComponent` | 删除组件 | 删除前读取层级；安全/恢复检查 | 组件树、编译和引用读回 | 响应/读回证据 |
| `ReparentComponent` | 改本地 SceneComponent 附着父级 | `ComponentName`、`NewParentName`；禁止循环，根改用 `SetRootComponent` | `GetComponents`、编译 | 待补真实截图 |
| `SetRootComponent` | 设本地 SceneComponent 为根 | 目标须本地 SceneComponent | `GetComponents`、编译 | 响应/读回证据 |
| `DuplicateComponent` | 复制本地 SCS 组件及模板属性 | 源/新名称、父级；实际本地根不可复制 | 组件树和属性读回 | 响应/读回证据 |
| `GetComponentProperties` | 分页读可编辑组件模板属性 | 蓝图、组件名、分页参数 | 作为修改前后基线 | 响应/读回证据 |
| `GetClassDefaults` | 分页读可编辑 CDO 默认值 | 蓝图、分页参数 | 作为 Class Defaults 写入基线 | 响应/读回证据 |
| `SetClassDefaults` | 原子写 CDO（含继承）默认值 | `Properties` 为区分大小写的 UE 文本；须先编译过 | `GetClassDefaults`、编译 | 待补真实截图 |
| `CreateBlueprint` | 创建支持的 Blueprint 资产 | `AssetPath`、父类等定义 | 概览、编译、必要时保存 | 响应/读回证据 |
| `ReparentBlueprint` | 改 Blueprint 类父类 | 先 `bDryRun`；层级变化/接口冲突须 `bAllowPotentialDataLoss` | 响应中的旧/新父类与编译报告，重开 | 待补真实截图 |
| `CompileBlueprint` | 编译并返回真实状态/诊断 | `BlueprintPath` | 状态为 `UpToDate` 或 `Warning`，再查诊断 | 响应/读回证据 |
| `SaveAsset` | 显式保存资产 | 目标资产路径 | 成功响应后重开/从磁盘重载 | 响应/读回证据 |
| `OpenAsset` | 打开资产编辑器 | 资产路径 | 编辑器/后续截图工具响应 | 响应/读回证据 |
| `CloseAsset` | 关闭资产编辑器 | 资产路径 | 后续打开状态或响应 | 响应/读回证据 |
| `ReloadBlueprintFromDisk` | 从磁盘重载 Blueprint | 默认 dry-run；丢弃内存修改须批准 | 重载后概览/图详情读回 | 响应/读回证据 |
| `DeleteAsset` | 安全删除资产 | Package、引用/Undo/Redo 安全检查 | Asset Registry 与磁盘共同核验 | 响应/读回证据 |
| `RenameAsset` | 重命名/移动资产 | 源与目标路径 | Asset Registry 读回、引用检查 | 响应/读回证据 |
| `DuplicateAsset` | 复制资产 | 源与目标路径 | 新路径概览/打开验证 | 响应/读回证据 |
| `CaptureGraphScreenshot` | 截取真实已初始化图表编辑器画面 | 先 `OpenAsset`；图、视图/尺寸参数 | 检查返回文件与视图 Token | 可视截图工具 |

## 重点教程：默认值、签名、组件与继承

### 1. `ModifyVariable`：声明默认值，不是节点值

先用 `GetBlueprintOverview` 确认 `VariableName` 是本蓝图声明的成员变量。再只提交需要变化的字段：`TypeName`、`DefaultValue`、`NewName`、`bInstanceEditable`、`Category` 或 `Tooltip`。`DefaultValue` 使用 UE 文本格式；空 `Category` 恢复 UE 默认分类，空 `Tooltip` 删除提示。变量被引用时不能改类型；有 RepNotify 的变量不能通过该工具非交互重命名。写后重新读取概览并编译，确认 `ChangedFields`；要持久化再 `SaveAsset`。

### 2. `ModifyFunctionSignature`：函数参数不是普通节点 Pin

先以 `GetGraphDetail` 获取精确 `FunctionGraphGuid` 和 Entry/Result 的稳定 `PinId`，不要用显示名称代替 ID。选择唯一的 `Operation`：`AddInput`、`AddOutput`、`RenamePin`、`RemovePin` 或 `ChangePinType`。默认 `bDryRun=true`；审阅调用者、派生类、Delegate、Override、接口与 AnimGraph blocker 后，写入还须 `bApproveReferenceUpdates=true`。删除参数或改参数类型还须 `bAllowPotentialDataLoss=true`。工具会事务性更新可安全处理的调用点、编译受影响蓝图，失败会回滚；最后读回签名和调用者，不会自动保存。

### 3. `SetPinDefaults`：只改未连接输入 Pin 的字面量

从 `GetGraphDetail` 取得 `NodeGuid`，传入非空 `PinDefaults` 映射，例如 `{\"Duration\": \"2.0\", \"InString\": \"Hello\"}`。对象/Class Pin 需要完整对象路径。该工具不改变节点、连线或图结构；若输入 Pin 已连接，运行时会忽略它的默认值，应先确认连线是否存在。写后用图详情逐个核对 Pin 默认值；结构变化改用 `ApplyGraphPatch`。

### 4. `SetComponentProperties`：改 SCS 模板，不改实例

先 `GetComponents` 取得本地 SCS `ComponentName`，建议再用 `GetComponentProperties` 建立属性基线。把 `Properties` 作为“属性名 → UE 文本值”映射，例如 `RelativeLocation: \"(X=0,Y=0,Z=50)\"`、`bHiddenInGame: \"True\"`。工具只改此 Blueprint 添加的组件模板；任一属性不合法，整次调用回滚并给出属性名与格式提示。写后读回属性、编译，最后按需保存。

### 5. `SetClassDefaults`：改 CDO，可覆盖继承属性

先运行过一次 `CompileBlueprint`，再 `GetClassDefaults` 找到可编辑的、**大小写敏感**的反射属性名。传入 `Properties` 的 UE 文本映射，如 `bReplicates: \"True\"` 或 `JumpMaxCount: \"2\"`。它改的是该 Blueprint CDO，作用于以后放置/生成且未覆写的实例；不会追溯改变已存在实例。工具能访问父 C++/父 Blueprint 的属性，这是 `ModifyVariable` 做不到的。写后读回 CDO、编译并按需保存。

### 6. `ReparentBlueprint` 与 `ReparentComponent`：两个“父级”完全不同

`ReparentBlueprint` 改的是类继承：传 `NewParentClassPath`（原生完整路径、唯一已加载短名或 Blueprint 生成类）。先 `bDryRun=true` 检查旧/新类、层级风险和重复继承接口；新父类不是旧父类的子类、旧父类缺失或接口冲突时，必须明确 `bAllowPotentialDataLoss=true`。它会刷新图、编译，若编译失败尝试 Undo 回滚；读回父类和编译报告后，再重开验证。

`ReparentComponent` 改的是同一 Actor Blueprint 内本地 **SceneComponent** 的附着树：传组件名与新的 `NewParentName`。不能把节点挂到自身/子孙节点下；非 SceneComponent 不可改；实际本地根组件必须用 `SetRootComponent`。成功后用 `GetComponents` 检查 Parent，并编译验证。

## 推荐操作流程

所有写操作都使用同一顺序：

1. **检查** — 确定精确资产、图、稳定 GUID、Pin、成员或组件。
2. **搜索** — 通过 `SearchGraphNodes` 获取合法 `SpawnerId`。
3. **预览** — 工具支持时使用 `bDryRun=true`，检查受影响资产、规范化操作、限制和 blocker。
4. **应用** — 只发送一次已审阅操作，不编造 GUID 或 `SpawnerId`。
5. **读回** — 再次查询目标图或资产并比较预期状态。
6. **编译** — 获取权威状态、错误和警告。
7. **显式保存** — 只有用户需要持久化时才调用 `SaveAsset`。

Dry-run 可能报告 Action Registry 节点、生成 Pin、类型提升或精确修改后布局无法在不写入时证明。这是预览能力的真实限制，不是自动写入许可；应用前仍须核对当前图和精确的 `SearchGraphNodes` 返回结果。

## 教程：创建计算函数

下面的示例创建：

```text
FinalScore = (BaseScore + Bonus) × Multiplier
```

1. 用 `CreateBlueprint` 创建 Actor 蓝图。
2. 用 `CreateFunction` 创建 `CalculateScore`，包含三个 `double` 输入和一个 `double` 输出。
3. 用 `GetGraphDetail` 读取新函数并记录 Entry/Result 节点 GUID。
4. 在这个函数图中搜索 `Add` 和 `Multiply`，使用返回的 `Operator:Add` 和 `Operator:Multiply`。
5. 预览一个同时包含两个节点和五条数据连接的 `ApplyGraphPatch`。
6. 对完全相同的 Patch 执行写入，使用原生 `double` 类型提升和自动 `Balanced` 布局。
7. 读回四个节点和六条连线，编译并显式保存。

最终图应形成一个从左到右的单一连通组件：无节点重叠、无反向边、没有任何编造的节点 ID。

## 正常编辑器截图与业务规则修改对比

以下图片直接截取自真实 UE 5.2 Blueprint Editor 窗口，没有使用会导致 Slate Brush 缺失和黑白方块伪影的离屏 Graph PNG 路径。

### 正常图表示例

已保存的 `CalculateScore` 教程函数包含 Action Registry 创建的 Add 与 Multiply 节点、完整数据/执行连线、成功编译和自动布局。

![正常 Blueprint Editor 中的 CalculateScore](./Images/Tutorial/calculate-score.png)

### 复杂正式业务流程全图

`EvaluateOrderBatchV2` 是一个包含 71 个节点的订单结算函数。同一张连通图中完成订单行累计、客户等级与首单折扣、地区运费、税费、人工复核决策、积分与风险分数，以及最终结算结果装配。

![正常 Blueprint Editor 中的完整订单结算流程](./Images/Tutorial/order-settlement-complex.png)

### 修改 1：Region 1 增加首单免运费

修改前：Region 1 直接执行 `ShippingCents = 1000`，首单判断分支不在执行路径中。

![Region 1 增加首单免运费之前](./Images/Tutorial/order-region1-before.png)

修改后：Region 1 先进入 `FirstOrder` 分支；首单执行 `ShippingCents = 0`，非首单保留原来的 `ShippingCents = 1000` 规则。

![Region 1 增加首单免运费之后](./Images/Tutorial/order-region1-after.png)

### 修改 2：积分基数排除运费

修改前：`PointsEarned = TotalCents / 100`。

![积分排除运费之前](./Images/Tutorial/points-before.png)

修改后：`PointsEarned = (TotalCents - ShippingCents) / 100`。同一视口可直接看到新增 Subtract 节点，以及保留的 Divide 和 Result 节点。

![积分排除运费之后](./Images/Tutorial/points-after.png)

### 修改 3：首单折扣由固定值改为小计 5%

修改前：函数通过带零默认值的同类型 Add 节点传递固定折扣。

![修改首单折扣规则之前](./Images/Tutorial/discount-before.png)

修改后：固定值路径替换为 `SubtotalCents / 20`，得到小计 5% 折扣，同时保持相同函数输出和图表画幅。

![修改首单折扣规则之后](./Images/Tutorial/discount-after.png)

三组对比展示的都是真实节点、默认值或连线变化，不是只移动节点或调整布局。对比图使用固定视口与稳定节点身份，并完成图表读回、成功编译和显式保存。若 dry-run 无法在不写入时证明生成 Pin 状态，则先核对现有 GUID 与连线，再执行单事务修改并在修改后再次验证。这些示例是受控的正式项目风格业务流程，不含客户项目数据。

## 查找资产与图表

- 用 `ListBlueprints` 分页查找资产。
- 用 `GetBlueprintOverview` 查看图、变量、函数、组件、接口和父类。
- 用 `ListBlueprintMembers` 查看函数、自定义事件、事件分发器和局部变量。
- 图表修改前后都用 `GetGraphDetail` 获取 GUID、Pin、默认值、连线和坐标。
- 用 `GetCompileErrors` 或 `CompileBlueprint` 获取权威诊断。

始终使用插件返回的精确资产路径。大型图应分页读取，不要根据截图推断结构。

## 安全添加图表逻辑

使用 Unreal 英文动作名搜索，例如 `Branch`、`For Loop`、`For Each Loop`、`Switch on Int`、`Switch on Name`、`Switch on String`、`Add`、`Subtract`、`Multiply` 或 `Divide`。

`ApplyGraphPatch` 支持：

- 带临时 ID 和 Action Registry `SpawnerId` 的 `Nodes`。
- `Connections` 与 `Disconnections`。
- 通过稳定 Node GUID 定位的 `RemoveNodes`。
- 通过稳定 Node GUID 和绝对图坐标定位的 `MoveNodes`。
- 可选的 `PinDefaults`、`PromotedType`、布局范围和布局风格。

整个 Patch 只使用一个事务。任一步失败都会尝试自动回滚；若工具无法证明目标图已恢复，会明确报告恢复失败。此时应停止后续写入，先用 `GetGraphDetail` 核对实际状态，不得在未知状态上直接重试。

## 成员、函数、Struct 与 Enum

- 变量工具管理类型、默认值、Category、Tooltip 和安全删除。
- 函数工具创建、重命名、执行受支持的签名修改，或安全删除用户函数。
- 局部变量动作同时绑定函数图和声明 GUID。
- Struct/Enum 修改使用稳定字段或条目标识，并报告引用影响。
- 可能丢失数据或改变 Enum 序列化值语义的操作需要独立显式批准。

函数重命名和签名修改默认先分析影响。遇到不支持的 Override、不完整引用扫描、不透明 Delegate 绑定、未加载派生类或无法安全恢复的状态时会 fail-closed。

### 函数重命名：真实保存、重开验证截图

下面两张图来自真实 UE 5.2 编辑器会话的 `RenameFunction` 验证：安全检查、引用更新、编译、显式保存后关闭并重新打开资产。它们是历史验证证据，不替代每次改名时应执行的 dry-run、读回和编译。

![RenameFunction 后的函数定义，真实 UE5.2 保存重开验证](./Images/Tutorial/function-rename-base.png)

![RenameFunction 后的调用者，真实 UE5.2 保存重开验证](./Images/Tutorial/function-rename-caller.png)

### 当前运行时函数、事件与绑定事件截图

以下三张图来自本次真实 UE 5.2 编辑器会话：先完成 `initialize`，运行时 `tools/list` 返回 55 个工具，再通过 `OpenAsset` 和 `CaptureGraphScreenshot` 截取。它们证明当前插件能读取并截取对应的真实图表目标；是否安全完成修改仍必须以工具响应、修改后读回、编译、显式保存和重开结果为准。

![包含丰富输入输出参数的函数签名与实现](./Images/Tutorial/function-signature-overview.png)

![组件事件辅助函数图](./Images/Tutorial/component-event-graph.png)

![Actor 生命周期、组件绑定事件与可变 Pin 流程](./Images/Tutorial/event-graph-overview.png)

## 组件与类默认值

添加、重命名、复制、重设父级、设置根组件、修改模板属性或删除组件前，先读取当前组件层级。类默认值工具读取或修改支持的 Blueprint 默认值，不会把实例值冒充为类默认值。

## 资产生命周期

资产工具可以创建、复制、重命名、重设父类、编译、打开、关闭、重载、删除和保存支持的蓝图资产。

- 编译成功不代表资产已经持久化。
- 磁盘重载默认 dry-run，丢弃内存状态前必须显式批准。
- 破坏性删除比普通图修改具有更严格的 Undo/Redo 和 Package 安全要求。
- 只有 UObject、Asset Registry 和磁盘检查一致时才能报告成功。

## 截取图表

先调用 `OpenAsset`，再使用 `CaptureGraphScreenshot`。优先选择文字、Pin 和连线可读的全图或节点聚焦画面。需要可比较画幅时，复用返回的视图位置、缩放、尺寸和 Viewport Token。

截图工具要求真正初始化的蓝图图表编辑器控件。完全无界面或尺寸为零的 Graph Widget 会被拒绝，不会生成误导性的空白图。

## 待补真实截图矩阵

下表是教程待补证据清单，**不是已完成截图的声明**。应使用真实 Editor 会话，在读回、编译、显式保存与重开核验后补入；未补齐前，以工具响应和读回结果为证据。

| 场景 | 应证明的事实 | 建议工具链 | 当前状态 |
|---|---|---|---|
| 函数签名与事件图 | 函数 Entry/Result 参数、绑定事件和可变 Pin 的真实可视目标 | 概览/成员 → 图详情 → 编译；修改时再补 Before/After | 已有本次 UE5.2 运行时截图；修改证据仍以读回为准 |
| 组件模板属性 | SCS 模板属性写入，非关卡实例覆盖 | `GetComponents` → `GetComponentProperties` → `SetComponentProperties` → 读回/编译 | 待补 |
| Class Defaults | CDO 改动与成员变量默认值的区别 | 编译 → `GetClassDefaults` → `SetClassDefaults` → 读回/重开 | 待补 |
| 蓝图父类 | dry-run 风险、明确批准、编译后父类变化 | `GetBlueprintOverview` → `ReparentBlueprint` → 编译/重开 | 待补 |
| 组件父子级 | SceneComponent 附着变化、根组件边界 | `GetComponents` → `ReparentComponent` 或 `SetRootComponent` → 编译 | 待补 |
| Struct | 字段 GUID、引用影响与数据损失批准 | `GetUserDefinedStruct` → dry-run → 修改 → 读回/编译 | 待补 |
| Enum | 条目整数值、序列化语义批准与读回 | `GetUserDefinedEnum` → dry-run → 修改 → 读回/编译 | 待补 |

## 设置

打开 **项目设置 → 插件 → MCP Blueprint**：

- **Port** — 默认 `8766`；被占用时最多尝试后续九个端口。
- **Auto Start** — 随编辑器启动服务器。
- **Auto Compile After Modify** — 支持的写操作完成后自动编译。
- **Request Timeout Seconds** — 普通工具超时。
- **Compile Timeout Seconds** — 编译类工具独立超时。

## 当前限制

- 目标平台为 Windows、macOS 和 Linux Editor；不提供 Runtime/Shipping 模块。各平台二进制以 Fab 官方构建结果作为发布门禁。
- 暂无“调用任意 Blueprint 函数并返回运行结果”的通用工具。
- Custom Event 和 Event Dispatcher 的签名修改能力尚未达到普通函数签名修改的完整程度。
- 某些 Blueprint Action 类型存在无法完整扫描或事务恢复的状态，此时会 fail-closed。
- 截图需要真实初始化的 Blueprint Graph Editor 控件。

## 后续开发与完善方向

后续准备推进：

1. 有界、隔离的通用 Blueprint 运行调用与结果读回能力。
2. 带引用影响分析和回滚的 Custom Event / Event Dispatcher 安全签名编辑。
3. 更深入的图生命周期、调用者/引用、继承和未加载资产影响检查。
4. 支持更多 Blueprint 资产类型和更复杂的原生 Action Registry 节点。
5. 补充连接、dry-run、批准、持久化、Undo 和拒绝场景的完整图文教程。

详细优先级和验收证据以 AIHub 为准。以上是开发方向，不承诺具体发布日期。

## 故障排查

- **无法连接：** 使用工具栏显示的实际端点，配置端口可能已被占用。
- **找不到工具：** 刷新 `tools/list`，不要依赖旧缓存。
- **写入被拒绝：** 阅读所有 blocker 和批准参数要求，重试前重新读取目标。
- **编译失败：** 获取编译诊断，根据真实蓝图错误修复后再保存。
- **图表结果异常：** 对比前后的 `GetGraphDetail`，适用时执行 Undo。
- **插件提示重建：** 安装与引擎小版本完全匹配的包。
- **项目中找不到插件：** 核对插件是否安装在该引擎的 `Engine/Plugins/Marketplace`；不要用添加项目依赖的方式绕过。
- **UE 5.5 提示项目文件过期，原因是已安装的 MCPBlueprint 不在项目描述符中：** 选择 **Not Now（暂不）**。引擎插件仍会自动加载；选择 **Update（更新）** 会把 MCPBlueprint 写入项目 `.uproject`，形成当前安装模式明确要避免的项目侧依赖。

## 支持

如有问题或反馈，请使用[统一联系页面](https://mengzhishanghun.github.io/mengzhishanghun/contact/)。
