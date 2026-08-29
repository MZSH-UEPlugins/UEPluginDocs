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

## 功能概览

当前版本注册 55 个工具：

| 能力 | 数量 | 工具 |
|---|---:|---|
| 发现与读取 | 6 | `ListBlueprints`、`GetBlueprintOverview`、`ListBlueprintMembers`、`GetGraphDetail`、`SearchGraphNodes`、`GetCompileErrors` |
| User Defined Struct 与 Enum | 6 | `CreateUserDefinedStruct`、`GetUserDefinedStruct`、`ModifyUserDefinedStruct`、`CreateUserDefinedEnum`、`GetUserDefinedEnum`、`ModifyUserDefinedEnum` |
| 变量、函数、事件与 Pin | 17 | `AddVariable`、`ModifyVariable`、`RemoveVariable`、`CreateFunction`、`RenameFunction`、`ModifyFunctionSignature`、`RemoveFunction`、`CreateCustomEvent`、`AddEventNode`、`AddBoundEvent`、`AddLocalVariable`、`RemoveLocalVariable`、`AddNodePin`、`RemoveNodePin`、`AddInterface`、`AddEventDispatcher`、`RemoveEventDispatcher` |
| 图表编辑与布局 | 4 | `ApplyGraphPatch`、`SetPinDefaults`、`FormatGraph`、`AddCommentBox` |
| 组件与默认值 | 11 | `GetComponents`、`AddComponent`、`SetComponentProperties`、`RenameComponent`、`RemoveComponent`、`ReparentComponent`、`SetRootComponent`、`DuplicateComponent`、`GetComponentProperties`、`GetClassDefaults`、`SetClassDefaults` |
| 资产生命周期与截图 | 11 | `CreateBlueprint`、`ReparentBlueprint`、`CompileBlueprint`、`SaveAsset`、`OpenAsset`、`CloseAsset`、`ReloadBlueprintFromDisk`、`DeleteAsset`、`RenameAsset`、`DuplicateAsset`、`CaptureGraphScreenshot` |

主要能力包括：

- 大型资产、成员、图表和 Pin 读取的稳定有界分页。
- 使用 Action Registry 返回的节点 ID，而不是编造节点类型。
- 单事务图表 Patch：创建、连接、断开、删除、移动、默认值和布局；失败时尝试自动回滚，恢复无法证明会明确报告。
- `Balanced`、`Straight`、`Compact` 三种确定性布局风格。
- 局部和全图布局中的原生 Comment Box 包围关系保护。
- 高风险修改的 dry-run 影响分析和显式批准门禁。
- 事务回滚，以及 Unreal 支持范围内的单步 Undo。
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

普通编辑工具只标脏，不会静默保存；需要保留改动时使用 `SaveAsset`。下表说明每个工具的用途和用户需要注意的边界。

| 工具 | 具体功能 | 必要注意事项 |
|---|---|---|
| `ListBlueprints` | 分页查找项目中的 Blueprint 资产。 | 大型项目请用 `PathFilter` 和分页缩小范围。 |
| `GetBlueprintOverview` | 查看蓝图的父类、图、变量、组件和接口概览。 | 需要提供准确的 `BlueprintPath`。 |
| `ListBlueprintMembers` | 分页查看函数、事件、分发器和局部变量。 | 大型蓝图请使用分页。 |
| `GetGraphDetail` | 查看指定图的节点、Pin、连线和布局信息。 | 可用 `GraphName` 限定目标图。 |
| `SearchGraphNodes` | 按英文动作名查找可创建的节点。 | 后续创建节点时只能使用返回的 `SpawnerId`。 |
| `GetCompileErrors` | 获取蓝图当前的编译诊断。 | 用于处理编辑器报告的错误或警告。 |
| `CreateUserDefinedStruct` | 在 `/Game` 下创建用户定义 Struct。 | 创建后不会自动保存。 |
| `GetUserDefinedStruct` | 查看 Struct 的字段定义。 | 大型 Struct 可分页读取。 |
| `ModifyUserDefinedStruct` | 添加、调整或删除 Struct 字段。 | 先使用 `bDryRun`；可能造成数据丢失时必须显式批准。 |
| `CreateUserDefinedEnum` | 在 `/Game` 下创建用户定义 Enum。 | 创建后不会自动保存。 |
| `GetUserDefinedEnum` | 查看 Enum 的条目、数值和显示名。 | 大型 Enum 可分页读取。 |
| `ModifyUserDefinedEnum` | 添加、调整或删除 Enum 条目。 | 先使用 `bDryRun`；改变序列化含义时必须显式批准。 |
| `AddVariable` | 为蓝图添加成员变量。 | 需要蓝图路径、变量名和 `TypeName`。 |
| `ModifyVariable` | 修改成员变量的类型、默认值、名称、可编辑性、分类或提示。 | 被引用的变量不能改类型；带 RepNotify 的变量不能通过此工具改名。 |
| `RemoveVariable` | 删除成员变量。 | 有引用或无法安全恢复时会被阻止。 |
| `CreateFunction` | 创建蓝图函数及其参数。 | 需要蓝图路径、函数名和参数定义。 |
| `RenameFunction` | 重命名用户函数，并更新可安全更新的引用。 | 先使用 dry-run；更新引用必须显式批准。 |
| `ModifyFunctionSignature` | 添加、重命名、删除或改类型函数参数。 | 默认 dry-run；删除参数或改类型可能造成数据丢失，须显式批准。 |
| `RemoveFunction` | 删除用户函数。 | 有引用或无法安全恢复时会被阻止。 |
| `CreateCustomEvent` | 在 Event Graph 中创建自定义事件。 | 目标必须是实际的 Event Graph。 |
| `AddEventNode` | 添加受支持的事件节点。 | 需指定蓝图和事件类型。 |
| `AddBoundEvent` | 为组件或委托添加绑定事件。 | 组件和委托必须能被解析。 |
| `AddLocalVariable` | 在函数图中添加局部变量。 | 需要函数名、变量名和类型。 |
| `RemoveLocalVariable` | 删除函数局部变量。 | 有引用或无法安全恢复时会被阻止。 |
| `AddNodePin` | 为支持可变 Pin 的节点增加 Pin。 | 先从图表详情取得目标节点标识。 |
| `RemoveNodePin` | 删除节点上可编辑的用户 Pin。 | 需要准确定位目标 Pin。 |
| `AddInterface` | 为蓝图添加接口。 | 使用接口类路径。 |
| `AddEventDispatcher` | 添加事件分发器。 | 指定名称和签名。 |
| `RemoveEventDispatcher` | 删除事件分发器。 | 有引用或无法安全恢复时会被阻止。 |
| `ApplyGraphPatch` | 在一次操作中创建、连线、断线、删除、移动节点，设置默认值并布局。 | 先搜索节点；支持 dry-run；失败时会尝试回滚。 |
| `SetPinDefaults` | 修改现有节点输入 Pin 的字面量。 | 已连线的输入 Pin 在运行时会忽略字面量。 |
| `FormatGraph` | 用 `Balanced`、`Straight` 或 `Compact` 整理图表布局。 | 可按图或范围指定布局。 |
| `AddCommentBox` | 添加原生 Comment Box。 | 指定文字和范围或目标节点。 |
| `GetComponents` | 查看 Actor Blueprint 的组件树、父级和模板信息。 | 仅适用于 Actor Blueprint。 |
| `AddComponent` | 添加本地 SCS 组件。 | 指定组件类、名称和可选父组件。 |
| `SetComponentProperties` | 修改本蓝图 SCS 组件模板属性。 | 属性使用 UE 文本格式；任一属性无效时整次修改回滚。 |
| `RenameComponent` | 重命名本地 SCS 组件并更新成员引用。 | 新名称必须唯一。 |
| `RemoveComponent` | 删除组件。 | 有引用或无法安全恢复时会被阻止。 |
| `ReparentComponent` | 修改本地 SceneComponent 的附着父级。 | 不能形成循环；根组件请用 `SetRootComponent`。 |
| `SetRootComponent` | 将本地 SceneComponent 设为根组件。 | 目标必须是本地 SceneComponent。 |
| `DuplicateComponent` | 复制本地 SCS 组件及模板属性。 | 实际本地根组件不能复制。 |
| `GetComponentProperties` | 分页查看可编辑的组件模板属性。 | 需要蓝图路径、组件名和分页参数。 |
| `GetClassDefaults` | 分页查看可编辑的 CDO 默认值。 | 大型属性集请使用分页。 |
| `SetClassDefaults` | 修改蓝图 CDO（含继承属性）的默认值。 | 属性名区分大小写，值使用 UE 文本格式。 |
| `CreateBlueprint` | 创建受支持类型的 Blueprint 资产。 | 指定资产路径和父类等定义；创建后不会自动保存。 |
| `ReparentBlueprint` | 更改 Blueprint 类的父类。 | 先使用 `bDryRun`；层级或接口冲突可能需要 `bAllowPotentialDataLoss`。 |
| `CompileBlueprint` | 编译指定蓝图并返回诊断。 | 用于处理编辑器报告的错误或警告。 |
| `SaveAsset` | 将资产的当前改动明确保存到磁盘。 | 普通编辑不会自动保存。 |
| `OpenAsset` | 打开资产编辑器。 | 截图前必须先打开资产。 |
| `CloseAsset` | 关闭资产编辑器。 | 关闭前请先保存需要保留的改动。 |
| `ReloadBlueprintFromDisk` | 用磁盘版本替换当前内存中的 Blueprint。 | 默认 dry-run；丢弃未保存改动必须显式批准。 |
| `DeleteAsset` | 删除资产。 | 删除具有更严格的引用和恢复安全检查。 |
| `RenameAsset` | 重命名或移动资产。 | 指定源路径和目标路径。 |
| `DuplicateAsset` | 复制资产。 | 指定源路径和目标路径。 |
| `CaptureGraphScreenshot` | 截取已打开蓝图图表的画面。 | 先 `OpenAsset`；图表控件必须已初始化。 |

## 重点教程：默认值、签名、组件与继承

### 1. `ModifyVariable`：声明默认值，不是节点值

先用 `GetBlueprintOverview` 找到本蓝图声明的目标变量。只传入需要变化的字段：`TypeName`、`DefaultValue`、`NewName`、`bInstanceEditable`、`Category` 或 `Tooltip`。`DefaultValue` 使用 UE 文本格式；空 `Category` 恢复 UE 默认分类，空 `Tooltip` 删除提示。变量被引用时不能改类型；带 RepNotify 的变量不能通过此工具非交互改名。需要保留改动时再使用 `SaveAsset`。

### 2. `ModifyFunctionSignature`：函数参数不是普通节点 Pin

使用 `GetGraphDetail` 找到函数图及 Entry/Result 的所需标识；不要用显示名称代替这些标识。选择一个操作：`AddInput`、`AddOutput`、`RenamePin`、`RemovePin` 或 `ChangePinType`。默认 `bDryRun=true`；更新引用时必须设置 `bApproveReferenceUpdates=true`。删除参数或改参数类型还须设置 `bAllowPotentialDataLoss=true`。工具会处理可安全更新的调用点，失败时尝试回滚；不会自动保存。

### 3. `SetPinDefaults`：只改未连接输入 Pin 的字面量

从 `GetGraphDetail` 取得目标节点标识，传入非空 `PinDefaults` 映射，例如 `{\"Duration\": \"2.0\", \"InString\": \"Hello\"}`。对象/Class Pin 需要完整对象路径。该工具不改变节点、连线或图结构；若输入 Pin 已连接，运行时会忽略它的默认值。结构变化请改用 `ApplyGraphPatch`。

### 4. `SetComponentProperties`：改 SCS 模板，不改实例

先用 `GetComponents` 找到本地 SCS `ComponentName`。把 `Properties` 作为“属性名 → UE 文本值”映射，例如 `RelativeLocation: \"(X=0,Y=0,Z=50)\"`、`bHiddenInGame: \"True\"`。工具只改此 Blueprint 添加的组件模板；任一属性不合法，整次调用会回滚并给出属性名与格式提示。按需使用 `SaveAsset` 保存。

### 5. `SetClassDefaults`：改 CDO，可覆盖继承属性

使用 `GetClassDefaults` 找到可编辑且**大小写敏感**的反射属性名。传入 `Properties` 的 UE 文本映射，如 `bReplicates: \"True\"` 或 `JumpMaxCount: \"2\"`。它改的是该 Blueprint CDO，作用于以后放置/生成且未覆写的实例；不会追溯改变已存在实例。工具还能访问父 C++/父 Blueprint 的属性，这是 `ModifyVariable` 做不到的。按需使用 `SaveAsset` 保存。

### 6. `ReparentBlueprint` 与 `ReparentComponent`：两个“父级”完全不同

`ReparentBlueprint` 改的是类继承：传 `NewParentClassPath`（原生完整路径、唯一已加载短名或 Blueprint 生成类）。先使用 `bDryRun=true` 查看风险；新父类不是旧父类的子类、旧父类缺失或接口冲突时，必须明确 `bAllowPotentialDataLoss=true`。它会刷新图；若操作失败会尝试 Undo 回滚。

`ReparentComponent` 改的是同一 Actor Blueprint 内本地 **SceneComponent** 的附着树：传组件名与新的 `NewParentName`。不能把节点挂到自身或子孙节点下；非 SceneComponent 不可改；实际本地根组件必须用 `SetRootComponent`。

## 推荐操作流程

涉及结构或数据变化的写操作可按以下顺序进行：

1. **定位** — 确定目标资产、图、Pin、成员或组件。
2. **搜索** — 通过 `SearchGraphNodes` 获取合法 `SpawnerId`。
3. **预览** — 工具支持时使用 `bDryRun=true`，检查受影响资产、规范化操作、限制和 blocker。
4. **应用** — 使用工具返回的标识，不编造 `SpawnerId`。
5. **保存** — 只有需要保留改动时才调用 `SaveAsset`。

Dry-run 不能展示所有生成 Pin、类型提升或最终布局。这是预览能力的限制，不代表工具会自动写入；写入前仍应使用当前搜索结果中的 `SpawnerId`。

## 教程：创建计算函数

下面的示例创建：

```text
FinalScore = (BaseScore + Bonus) × Multiplier
```

1. 用 `CreateBlueprint` 创建 Actor 蓝图。
2. 用 `CreateFunction` 创建 `CalculateScore`，包含三个 `double` 输入和一个 `double` 输出。
3. 用 `GetGraphDetail` 找到新函数的 Entry/Result 节点标识。
4. 在这个函数图中搜索 `Add` 和 `Multiply`，使用返回的 `Operator:Add` 和 `Operator:Multiply`。
5. 预览一个同时包含两个节点和五条数据连接的 `ApplyGraphPatch`。
6. 对完全相同的 Patch 执行写入，使用原生 `double` 类型提升和自动 `Balanced` 布局。
7. 需要保留这项改动时，调用 `SaveAsset`。

最终图从左到右呈现 `BaseScore + Bonus` 后再乘以 `Multiplier` 的计算流程。

## 正常编辑器截图与业务规则修改对比

以下示例展示 Blueprint Editor 中常见的计算函数、完整业务图和规则调整前后的可见差异。它们用于帮助你理解节点、连线和默认值变化。

### 正常图表示例

`CalculateScore` 展示 Add 与 Multiply 节点如何组合为一个清晰的计算函数。

![正常 Blueprint Editor 中的 CalculateScore](./Images/Tutorial/calculate-score.png)

### 复杂正式业务流程全图

`EvaluateOrderBatchV2` 展示一个订单结算函数如何在同一张图中处理订单累计、折扣、运费、税费、积分、风险分数和最终结果。

![正常 Blueprint Editor 中的完整订单结算流程](./Images/Tutorial/order-settlement-complex.png)

### 修改 1：Region 1 增加首单免运费

修改前，Region 1 直接执行 `ShippingCents = 1000`，不区分是否为首单。

![Region 1 增加首单免运费之前](./Images/Tutorial/order-region1-before.png)

修改后，Region 1 增加 `FirstOrder` 分支：首单运费为 `0`，其他订单仍为 `1000`。

![Region 1 增加首单免运费之后](./Images/Tutorial/order-region1-after.png)

### 修改 2：积分基数排除运费

修改前：`PointsEarned = TotalCents / 100`。

![积分排除运费之前](./Images/Tutorial/points-before.png)

修改后：`PointsEarned = (TotalCents - ShippingCents) / 100`，新增 Subtract 节点以排除运费。

![积分排除运费之后](./Images/Tutorial/points-after.png)

### 修改 3：首单折扣由固定值改为小计 5%

修改前：`SubtotalCents × 0` 后再加固定值 `500`，因此输出始终是固定折扣 `500`。

![修改首单折扣规则之前](./Images/Tutorial/discount-before.png)

修改后：固定值路径改为 `SubtotalCents / 20`，即小计的 5% 折扣。

![修改首单折扣规则之后](./Images/Tutorial/discount-after.png)

修改逻辑前，先使用 dry-run（若工具支持）了解影响；涉及引用更新或数据丢失时，按工具要求显式批准。需要保留改动时使用 `SaveAsset`。

## 查找资产与图表

- 用 `ListBlueprints` 分页查找资产。
- 用 `GetBlueprintOverview` 查看图、变量、函数、组件、接口和父类。
- 用 `ListBlueprintMembers` 查看函数、自定义事件、事件分发器和局部变量。
- 用 `GetGraphDetail` 查看节点、Pin、默认值、连线和布局。
- 用 `GetCompileErrors` 或 `CompileBlueprint` 查看编辑器诊断。

使用插件返回的资产路径。大型图请分页查看，避免一次返回过多内容。

## 安全添加图表逻辑

使用 Unreal 英文动作名搜索，例如 `Branch`、`For Loop`、`For Each Loop`、`Switch on Int`、`Switch on Name`、`Switch on String`、`Add`、`Subtract`、`Multiply` 或 `Divide`。

`ApplyGraphPatch` 支持：

- 带临时 ID 和 Action Registry `SpawnerId` 的 `Nodes`。
- `Connections` 与 `Disconnections`。
- 通过节点标识定位的 `RemoveNodes`。
- 通过节点标识和图坐标定位的 `MoveNodes`。
- 可选的 `PinDefaults`、`PromotedType`、布局范围和布局风格。

整个 Patch 只使用一个事务，失败时会尝试自动回滚。如果工具提示恢复失败，请停止继续修改，重新打开资产确认当前状态后再处理。

## 成员、函数、Struct 与 Enum

- 变量工具管理类型、默认值、Category、Tooltip 和安全删除。
- 函数工具创建、重命名、执行受支持的签名修改，或安全删除用户函数。
- 局部变量操作需要函数图和变量声明标识。
- Struct/Enum 修改使用字段或条目标识，并会报告引用影响。
- 可能丢失数据或改变 Enum 序列化值语义的操作需要独立显式批准。

函数重命名和签名修改会先分析影响。遇到无法安全处理的 Override、Delegate 绑定、派生类或引用时，工具会拒绝操作，不会勉强修改。

### 函数重命名示例

下面两张图展示函数改名后的定义与调用点。改名前先使用 dry-run；工具要求更新引用时，必须显式批准。需要保留改动时使用 `SaveAsset`。

![RenameFunction 后的函数定义](./Images/Tutorial/function-rename-base.png)

![RenameFunction 后的调用者](./Images/Tutorial/function-rename-caller.png)

### 函数、事件与绑定事件示例

以下示例展示带输入输出参数的函数、组件事件辅助函数图，以及 Actor 生命周期、组件绑定事件和可变 Pin 的常见结构。

![包含丰富输入输出参数的函数签名与实现](./Images/Tutorial/function-signature-overview.png)

![组件事件辅助函数图](./Images/Tutorial/component-event-graph.png)

![Actor 生命周期、组件绑定事件与可变 Pin 流程](./Images/Tutorial/event-graph-overview.png)

## 组件与类默认值

添加、重命名、复制、重设父级、设置根组件、修改模板属性或删除组件前，先读取当前组件层级。类默认值工具读取或修改支持的 Blueprint 默认值，不会把实例值冒充为类默认值。

## 资产生命周期

资产工具可以创建、复制、重命名、重设父类、编译、打开、关闭、重载、删除和保存支持的蓝图资产。

- 磁盘重载默认 dry-run，丢弃内存状态前必须显式批准。
- 破坏性删除比普通图修改具有更严格的 Undo/Redo 和 Package 安全要求。

## 截取图表

先调用 `OpenAsset`，再使用 `CaptureGraphScreenshot`。可以截取完整图表、聚焦某个节点，或复用指定视口。分享图片前，请确认标题、Pin 和连线清晰可读。

截图工具要求已初始化的蓝图图表编辑器控件。未打开图表编辑器或图表控件尺寸为零时，截图请求会被拒绝。

## 设置

打开 **项目设置 → 插件 → MCP Blueprint**：

- **Port** — 默认 `8766`；被占用时最多尝试后续九个端口。
- **Auto Start** — 随编辑器启动服务器。
- **Auto Compile After Modify** — 支持的写操作完成后自动编译。
- **Request Timeout Seconds** — 普通工具超时。
- **Compile Timeout Seconds** — 编译类工具独立超时。

## 当前限制

- 支持 Windows、macOS 和 Linux Editor；不提供 Runtime/Shipping 模块。
- 暂无“调用任意 Blueprint 函数并返回运行结果”的通用工具。
- Custom Event 和 Event Dispatcher 的签名修改能力尚未达到普通函数签名修改的完整程度。
- 某些 Blueprint Action 无法安全处理时，工具会直接拒绝操作。
- 截图需要真实初始化的 Blueprint Graph Editor 控件。

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
