[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint

MCPBlueprint 是一个自包含的 Unreal Editor 插件，通过 HTTP MCP 向 AI 客户端提供蓝图发现、图表编辑、成员管理、组件管理、资产生命周期与编译反馈能力。插件支持 Unreal Engine 5.2 及以上版本，不依赖其他用户制作的插件；带预编译二进制的 Win64 Editor 插件包已覆盖验证 UE 5.2–5.8。

Fab 英文商品文案草稿、当前逐版本兼容矩阵、安装步骤、7 个完整工作流、隐私/网络说明、故障排查与截图素材见 [FAB_LISTING.md](./FAB_LISTING.md)。该文件不是最终提交产物。

## 连接方式

默认端点为 `http://127.0.0.1:8766/mcp`。若端口被占用，服务器会继续尝试后续端口，工具栏会显示实际端点。请在 AI 客户端中把该 URL 配置为 HTTP MCP 服务器。

服务器实现 `initialize`、`tools/list`、`tools/call` 和 `ping`。写操作在游戏线程执行；引擎支持时会纳入编辑器事务以便 Undo；修改只把资产标记为 Dirty，不会自动保存到磁盘。

当前注册的 55 个工具均由编辑器自动化 Schema/调用矩阵覆盖：逐一检查等价于 `tools/list` 的定义是否拥有可序列化且边界闭合的输入 Schema、非空且唯一的名称和非空描述，并在注册表边界验证未知字段与缺失必填字段均被拒绝。零必填工具必须先显式归类为安全只读，才允许以空对象调用；不完整的操作专用 `oneOf`/`const` 分支同样会在真正执行前被拒绝。

请求必须使用 JSON-RPC `2.0`，且 `params` 与工具 `arguments` 必须是对象。浏览器风格的 `Origin` 仅允许 `localhost`、`127.0.0.1` 或 `[::1]`；非浏览器客户端可以不发送 `Origin`。

## 工具（55 个）

### 发现与读取

| 工具 | 用途 |
|---|---|
| `ListBlueprints` | 使用稳定且有上限的分页列出蓝图资产。 |
| `GetBlueprintOverview` | 读取图表、变量、函数、组件、接口与父类。 |
| `ListBlueprintMembers` | 使用统一稳定分页列出函数、Custom Event、Dispatcher 与局部变量。 |
| `GetGraphDetail` | 读取图表节点、Pin、默认值和连接。 |
| `SearchGraphNodes` | 使用 Unreal 英文标准名称搜索蓝图动作并返回稳定的 `SpawnerId`；标准查询包括 `Branch`、`For Loop`、`For Each Loop`、`Switch on Int/Name/String`、`Add/Subtract/Multiply/Divide`，以及函数作用域变量的 `Get`/`Set`。 |
| `GetCompileErrors` | 编译并返回蓝图权威状态与诊断。 |

### User Defined Struct 与 Enum

| 工具 | 用途 |
|---|---|
| `CreateUserDefinedStruct` / `GetUserDefinedStruct` / `ModifyUserDefinedStruct` | 创建、读取并按稳定字段 GUID 安全修改结构体。 |
| `CreateUserDefinedEnum` / `GetUserDefinedEnum` / `ModifyUserDefinedEnum` | 创建、读取并安全修改枚举项。 |

先用 `bDryRun=true` 审阅有界引用影响结果；删除结构体字段或修改字段类型随后必须显式传入 `bAllowPotentialDataLoss=true`，删除或重排枚举项必须显式传入 `bAllowSerializedValueSemanticChange=true`。枚举重命名只修改显示名并保留内部枚举项名称。修改操作会刷新并编译引用 Blueprint，报告受影响的 DataTable/DataAsset，纳入 Undo 并只标记 Dirty，绝不自动保存。创建会先完成有效性验证；失败的部分资产会被丢弃，也不会通知 Asset Registry。

### 变量、函数与事件

| 工具 | 用途 |
|---|---|
| `AddVariable` / `ModifyVariable` / `RemoveVariable` | 管理蓝图成员变量；`ModifyVariable` 还可事务化更新 `Category` 与 `Tooltip`。 |
| `CreateFunction` / `RenameFunction` / `ModifyFunctionSignature` / `RemoveFunction` | 创建、安全门禁重命名、安全修改可编辑用户函数签名或安全删除函数图。重命名与签名修改默认只做 dry-run 影响分析，更新引用前必须显式批准。 |
| `CreateCustomEvent` / `AddEventNode` / `AddBoundEvent` | 添加自定义事件、引擎事件、组件事件或关卡 Actor 事件。 |
| `AddLocalVariable` / `RemoveLocalVariable` | 添加或安全删除函数局部变量；添加后，在该变量所属的精确函数图中按名称搜索，以取得 Action Registry 支持的 `LocalGet` 与 `LocalSet` 动作。 |
| `AddNodePin` / `RemoveNodePin` | 为 Sequence、容器和 Switch 等节点添加或删除受支持的动态 Pin。 |
| `AddInterface` | 实现蓝图接口。 |
| `AddEventDispatcher` / `RemoveEventDispatcher` | 创建或安全删除多播事件分发器及其签名。 |

调用 `RenameFunction` 时，省略 `bDryRun`（或设为 `true`）只会返回稳定 Graph GUID、有界的本地/外部引用计数、受影响 Blueprint、未加载派生类与阻断项，不产生修改。正式执行必须同时传入 `bDryRun=false` 与 `bApproveReferenceUpdates=true`。首版会主动拒绝 override/受保护函数、RepNotify 函数、不完整引用扫描、未加载派生 Blueprint、`CreateDelegate` 绑定，以及在声明 Blueprint、已加载依赖或 Asset Registry 外部引用项中发现 AnimBlueprint/AnimGraph 状态的情况，即使普通节点引用计数为 0 也会阻断。该门禁采用 fail-closed，因为 UE 5.4+ 可能改写目前无法完整扫描或事务恢复的嵌套函数属性绑定。成功执行会保留 `FunctionGraphGuid`、确认旧名称引用归零、编译全部受影响 Blueprint、纳入 Undo、只标记 Dirty，绝不自动保存。

`ModifyFunctionSignature` 每次只接受 `AddInput`、`AddOutput`、`RenamePin`、`RemovePin` 或 `ChangePinType` 之一。声明必须同时用精确 `FunctionName` 与稳定 `FunctionGraphGuid` 定位；既有参数只能使用 `GetGraphDetail` 返回的 `PinId` 选择，名字仅供显示，不作为身份。省略 `bDryRun` 时只扫描普通本地/外部 Caller、Asset Registry 引用、派生类、Delegate/不透明绑定、编译基线、不兼容类型连接和阻断项，不修改资产。正式执行必须传入 `bDryRun=false` 与 `bApproveReferenceUpdates=true`；`RemovePin`、`ChangePinType` 还必须额外传入 `bAllowPotentialDataLoss=true`。成功执行会核验完整声明/Caller Pin 契约：重命名只允许名字变化，目标 PinId、完整 `FEdGraphPinType` 语义、默认值和连线必须保留；兼容的类型/默认值变化必须保留目标 PinId 与连线；新增必须在声明和 Caller 生成匹配 Pin 与 Caller 输入默认值；删除只允许移除已批准目标及其连线。任何非目标 PinId、方向、完整类型限定（category/subcategory/object/container/reference/const/weak）、字符串默认值、稳定 DefaultObject 路径、文本默认值、相对顺序或连接端点发生变化都会整笔回滚；输出 Pin 不承担 Caller 输入默认值契约。不兼容的已连接类型变化会被阻断，不会静默断线。之后重建普通调用节点、编译全部受影响 Blueprint、支持一次 Undo、只标记 Dirty，绝不自动保存。override/interface/RepNotify 函数、缺少 Generated/Skeleton Class、不可验证编译状态、已加载或未加载的派生 Blueprint、不完整扫描、非普通调用引用、`CreateDelegate` 以及 AnimBlueprint/AnimGraph 候选均按 fail-closed 拒绝。首个安全单元有意不包含 Pin 重排。

函数局部变量动作同时受函数图 GUID 和局部变量声明 GUID 约束。`SearchGraphNodes` 只会在声明图中返回 `LocalGet:<GraphGuidDigits>:<LocalVarGuidDigits>` 与 `LocalSet:<GraphGuidDigits>:<LocalVarGuidDigits>`；Setter 提供 `execute`、`then`、同名变量值输入和 `Output_Get`。必须把搜索返回的精确 ID 交给 `ApplyGraphPatch`；跨图、过期或伪造的 GUID 组合会 fail-closed，节点放置则复用原始 Action Registry Spawner，以保留 Unreal 原生局部作用域。

### 图表编辑

| 工具 | 用途 |
|---|---|
| `ApplyGraphPatch` | 只读预览，或在单个声明式事务中原子执行有上限的节点创建/删除、连接/断开、默认值、移动和布局修改。 |
| `SetPinDefaults` | 为现有输入 Pin 设置经过校验的字面量默认值。 |
| `FormatGraph` | 自动排列整张图或指定节点集合。 |
| `AddCommentBox` | 以可配置颜色、标题和边距把指定节点框入原生 Comment Box。 |

#### 专业蓝图布局

`FormatGraph` 使用确定性的弱连通区域分离、SCC 循环压缩、从左到右的分层拓扑、重心法交叉线优化、Slate 节点与 Pin 行尺寸及其有界回退、Pin-aware 纵向对齐，以及多区域装箱。`LayoutScope` 支持 `WholeGraph`、`ConnectedComponent` 和 `Selection`。`LayoutStyle` 支持 `Balanced`（默认通用间距）、`Straight`（更强的连线拉直和更多纵向空间）与 `Compact`（更小占用、较轻的拉直）；项目约定或 AI 技能规范只需选择预设，不必绑定算法内部权重。整图布局会保护注释框及其当前包围的节点。设置 `bDryRun=true` 时不会调用 `Modify()` 或开启编辑器事务，只返回规划位置，以及布局前后的重叠、反向边、交叉线、长连线、占用面积、`FlatEdgeRatio`、`AveragePinDeltaY` 和 `P95PinDeltaY` 指标。可选 Reroute 默认关闭，并受 `MaxRerouteNodes` 限制；其质量指标描述插入 Knot 前的原节点布局方案。

对 `Selection` 和 `ConnectedComponent`，已经包围目标节点的 Comment Box 只在本次局部规划中不再作为外部障碍；规划完成前会逐节点核对完整的原包围关系。节点必须留在每个原 Comment 的内容区域内、保持在无关 Comment 之外，并避开由 Slate 在无窗口条件下真实测得的换行标题高度及额外 8 Graph Units 安全边距；同框内未选中的节点仍是固定障碍，嵌套 Comment 也会同时约束。若原状态已经跨边界、多个容器不兼容、框内空间不足，或图中存在任何 Comment Box 时请求 `bInsertReroutes=true`，工具会 fail-closed，不改变图拓扑和 Undo 状态，因为新增 Knot 不在写入前 containment 计划中。结果返回 `PreservedCommentRelationshipCount`，dry-run 与 apply 使用同一套确定性约束计划。

`ApplyGraphPatch` 支持 `LayoutScope=Auto|CreatedNodes|ConnectedComponent|None`，并接受相同的 `LayoutStyle` 预设。应先设置 `bDryRun=true`，取得稳定、纯只读的计划：`OperationCounts`、规范化后的创建/删除/移动/连接/断连条目、`LayoutPlan` 与 `Blockers`。该分支不会调用 `Modify()`、不会开启事务、不会把 Package 标成 Dirty、不会改变 Undo 或 Asset Registry 状态，也不会编译 Blueprint。既有节点操作会直接做只读校验；Action Registry 节点创建、动态或 wildcard Pin、Unreal 原生自动转换/类型提升，以及修改后的精确布局无法在不变更图的前提下证明，因此会明确列为 blocker，绝不采用“先执行再 Undo”冒充预览。Blocker 表示预览刻意保持 fail-closed，并不表示可以跳过 `SearchGraphNodes`、写入时校验、读回、编译或 Undo 测试。

普通 AI 调用只提交节点逻辑、Pin 默认值和连接关系，省略 `Position` 与 `LayoutScope`，由 MCPBlueprint 默认 `Auto` 负责连接后的标准分层、Pin 顺序、节点间距和少交叉排版；只有用户明确要求固定手工布局时才使用 `Position` 或 `None`。MCP 驱动的节点放置默认必须避免重叠，且不能静默接受仍未解决的碰撞。显式提供 `Position` 的节点保持固定。布局、Reroute 或编译失败都会进入同一个图补丁事务回滚；写入结果会返回实际采用的范围、风格和布局质量指标。

UE 5.2 真实 MCP 验收覆盖了三种调用方式，全部省略 `Position` 与 `LayoutScope`：一次 Patch 创建并连接节点后，交叉从 1 降为 0、反向边从 2 降为 0；先创建节点、再用纯 `Connections` Patch 接线时，第二笔请求自动识别受影响连通组件并把反向边从 2 降为 0；复用已有 `BeginPlay` 事件再连接 `Print String` 时，反向边从 1 降为 0，Pin 高差从 552 降为 1。`Auto` 同样会处理复用节点、Move-only、断连端点和删除节点的存活邻居；普通数据输入的互斥连接会整笔拒绝，Exec 输入允许多来源，合法自动转换节点会纳入最终连接验证和布局。

![一次 Patch 自动成图](./Images/LayoutContract/01_OneShotAuto.png)

![分两次调用后由纯连接 Patch 自动重排](./Images/LayoutContract/02_TwoStageAuto.png)

![复用已有 BeginPlay 事件并自动接续布局](./Images/LayoutContract/03_ReusedEventAuto.png)

当新增业务块需要让出空间，或复杂图需要按阶段重新组织时，可使用 `Patch.MoveNodes` 有界、显式地移动既有节点。每项格式为 `{ "NodeGuid": "...", "Position": { "X": 1200, "Y": 600 } }`，坐标是图中的绝对坐标。该操作只改变位置，节点 GUID、类型、Pin、默认值和全部既有连线保持不变；所有移动与同一 `ApplyGraphPatch` 的编译门禁、失败回滚和单次 Undo 共用一个事务。应先通过 `GetGraphDetail` 读取 GUID 与当前坐标。未知或重复 GUID、非法/越界坐标以及超过共享 60 项补丁上限的请求会整体拒绝，不留下部分修改。

当 `Auto` 在已有实现中回退到 `CreatedNodes` 时，会先根据真实的外部前驱/后继连接，把每个新增连通块锚定到插入位置。如果新增块需要更多水平空间，只把右侧有界碰撞闭包沿 `+X` 整体刚体平移：闭包内所有节点使用同一个位移且 `Y` 不变，闭包外节点的 GUID、坐标和连接保持不变。若需要移动 Function Entry、破坏 Comment Box 包围关系、超过节点/迭代/距离上限，或产生任何新的反向边，整笔补丁会 fail-closed。显式 `LayoutScope=CreatedNodes` 则保留原有的“放到既有逻辑下方”语义，不移动旧节点。所有写工具仍然不会自动保存：应先读回并编译，再显式调用 `SaveAsset`；一次编辑器 Undo 会同时恢复新增节点和被平移的闭包。

### 布局压力证据与可读逻辑验收

以下 UE 5.2 截图保留为**布局压力测试证据**：它们只证明有界 `+X` 平移、一次 Undo 和保存/关闭/重开后的持久化；**不代表真实业务逻辑，也不通过产品验收**。102/108 节点场景不能再作为正常生产连线示例。后来补做的 112/114 节点连通场景同样未通过用户的产品级截图验收：无孤立节点、编译成功、保存重开和 Undo/Redo，都不能证明函数已在固定输入下被实际调用并产生正确业务结果。GetterSafety `01–05` 也只能作为实现/过程证据。不得复用这两类旧测试图作为下次验收基线。下一次隔离测试资产的真实业务门槛、运行时证明、逐次截图、否决条件和 Definition of Done，以[《蓝图业务流验收规范》](./GRAPH_WORKFLOW_ACCEPTANCE.md)为准。

![102 节点布局压力测试基线](./Images/LayoutShift/01_Baseline_102Nodes.png)

*仅为布局压力测试基线，不是连通的真实业务逻辑示例。*

![插入三节点算术链并局部让位](./Images/LayoutShift/03_Insert_ArithmeticChain_AutoShift.png)

*压力测试第 2 阶段（`103 → 106`）：9 个旧节点统一平移 `+633`、`DeltaY=0`。此图仅证明局部让位，不证明业务逻辑可读性。*

![持久化算术插入的压力测试局部图](./Images/LayoutShift/03_ArithmeticChain_Local.png)

*保存并重开后的压力测试视图；仅保留为局部平移证据，不是可读工作流示例。*

![第二处插入只移动更小闭包](./Images/LayoutShift/04_Insert_AddPair_AutoShift.png)

*压力测试第 3 阶段（`106 → 108`）：2 个旧节点统一局部平移 `+298`；编译、Undo、保存、关闭和重开通过。它不是业务逻辑验收图。*

![持久化双 Add 的压力测试局部图](./Images/LayoutShift/04_AddPair_Local.png)

*仅为保存/重开后的压力测试局部图；[第 1 阶段局部图](./Images/LayoutShift/02_Insert_Add_Local.png) 仍证明单个 Add 可在不移动旧节点时放入空隙。*

真实工作流的 100+ 节点验收必须具有连通的 Exec 与数据链。Branch、For Loop、For Each Loop、Switch，以及 Add/Subtract/Multiply/Divide 的结果必须进入后续赋值、判断或返回，不能只作为孤立节点存在。成员变量、函数参数或局部变量应在每个消费区域附近创建合法的 Get；不要把 Function Entry Pin 或远端 Getter 向全图星形扇出。临时计算值不能被盲目复制成 Get；应通过局部变量 Set/Get 对或有界 Reroute 保持原有语义。只有没有执行 Pin 的纯变量 Getter 才会按消费者自动拆分并放到使用位置附近；同一消费者旁的多个局部 Getter 会稳定纵向堆叠，并使用与图布局一致的 60 单位间距检查。找不到安全局部位置时，整笔 patch 失败并回滚。Validated Get 等带执行流语义的非纯 Getter 始终保持单节点和原有执行连接，不参与自动拆分。消费者局部重锚只在 `Auto` 与 `ConnectedComponent` 下执行；显式 `CreatedNodes` 保留原有的旧图下方放置方式，`None` 不触发任何隐式布局。`ApplyGraphPatch` 不会静默改写请求 patch 之外的旧逻辑或旧连线；`Auto` 仅允许改变上文明确报告的右侧有界闭包坐标。修改前先对完全相同的 Patch 执行 `bDryRun=true` 并逐项审阅 blocker；解决可审阅问题后再用 `bDryRun=false` 写入，随后仍必须完成原子事务、立即读回/编译和编辑器 Undo 验证，不能把 dry-run 当成写路径验收的替代品。

创建标准算术节点时，先用 Unreal 英文动作名 `Add`、`Subtract`、`Multiply` 或 `Divide` 调用 `SearchGraphNodes`，再把其返回的精确 `Operator:<Name>` 交给 `ApplyGraphPatch`，不要手工拼接 SpawnerId。在同一个 patch 中，把 Real/Double 来源 Pin 连接到 `A` 或 `B`，并为节点可选传入 `PromotedType: "double"` 或 `"real"`。Unreal Schema 会执行原生的连接驱动类型提升；全部连接完成后，工具再验证目标运算函数与 `A`、`B`、`ReturnValue` 均为 Real/Double，否则整笔 patch 回滚。不同引擎版本和 Pin 上下文可能在界面或序列化文本中显示为 **Float**、**Double** 或 **Real**；当前流程明确保证 Real/Double 形式，不承诺强制独立的单精度 Float 形式。

标准流程控制节点请使用 Unreal 英文名称搜索：`Branch`、`For Loop`、`For Each Loop`、`Switch on Int`、`Switch on Name`、`Switch on String`。循环节点来自真实的 StandardMacros Action Registry（例如 `Macro:/Engine/EditorBlueprintResources/StandardMacros.StandardMacros:ForLoop`），Branch 和三种标量 Switch 使用 Registry 支持的 `Native:K2Node_*` ID。必须使用搜索返回值；手工伪造宏或原生 ID 仍会被拒绝。节点放置后，编辑器界面标题可按 UE 当前语言本地化，但搜索名保持英文标准名称。

![由 SearchGraphNodes 结果创建的标准流程控制节点](./Images/Fab/FlowControl_StandardNodes.png)

*真实 UE 5.2 截图：已完成 Search → ApplyGraphPatch → 读回 → 编译 → 保存 → 关闭重开；六个标准节点彼此分离且无重叠。*

### Actor 蓝图组件与默认值

| 工具 | 用途 |
|---|---|
| `GetComponents` / `GetComponentProperties` | 读取本地 SCS 层级与组件模板的可编辑属性。 |
| `AddComponent` / `DuplicateComponent` / `RenameComponent` / `RemoveComponent` | 管理本地 SCS 组件。 |
| `ReparentComponent` / `SetRootComponent` | 修改本地 SceneComponent 层级。 |
| `SetComponentProperties` | 使用 UE 文本格式原子设置组件模板属性。 |
| `GetClassDefaults` / `SetClassDefaults` | 分页读取可编辑的 Class Default Object 属性，再原子设置经确认的值。 |

### 资产生命周期与截图

| 工具 | 用途 |
|---|---|
| `CreateBlueprint` | 在 `/Game` 下创建内存中的蓝图资产。 |
| `CompileBlueprint` | 编译并返回权威状态与诊断。 |
| `SaveAsset` | 显式保存独立 Blueprint、User Defined Struct 或 User Defined Enum 包；关卡蓝图和其他资产类型会被拒绝。 |
| `OpenAsset` / `CloseAsset` | 在蓝图编辑器中打开或关闭独立 Blueprint。 |
| `ReloadBlueprintFromDisk` | 先 dry-run，或在明确批准丢弃内存状态后，通过 Unreal 官方包重载 API 从已有磁盘文件重载一个已加载的独立 Blueprint。 |
| `DeleteAsset` / `RenameAsset` / `DuplicateAsset` | 管理独立蓝图资产。 |
| `ReparentBlueprint` | 预演或执行安全父类修改，拒绝继承循环并报告潜在数据丢失。 |
| `CaptureGraphScreenshot` | 返回 PNG，也可保存到 `Saved/MCPBlueprint/Screenshots` 下。默认清空选择，支持 `NodeGuid`、`bZoomToFit`、显式 `ViewLocation` + `Zoom` 或 `GraphRect` 导航。为兼容旧调用，`NodeGuid` 搭配省略/true 的 `bZoomToFit` 会先跳转节点再适配全图；显式 false 才使用节点 1:1 视图。响应返回实际变换、`ViewMode` 和稳定 `ViewportToken`。 |

`ViewportToken` 对相同 Graph、视图模式、实际视口变换和输出尺寸保持稳定，用于证明画幅一致，不承诺 PNG 字节完全相同：Slate hover、动画、字体、主题、DPI 与 GPU 状态仍是渲染瞬态。制作 Before/After 时应复用响应中的 `AppliedViewLocation`、`AppliedZoom`、`Width` 和 `Height`，并保持 `bClearSelection=true`。`GraphRect` 在不拉伸画面的前提下适配指定图坐标矩形，因此长轴精确，另一轴可能保留确定性的额外边距；若所需 Zoom 低于 `0.05`，工具会明确拒绝，不会静默裁切区域。

## 固定订单结算基线用例

该已纳管示例是一个**五节点可执行基线**，不是完整订单结算业务流，也不是 100+ 节点产品验收目标。

- 订单行结构体：`/Game/MCPBP_AutoTest/BusinessWorkflow/OrderSettlement/ST_OrderLine`
- 结果结构体：`/Game/MCPBP_AutoTest/BusinessWorkflow/OrderSettlement/ST_SettlementResult`
- Actor Blueprint：`/Game/MCPBP_AutoTest/BusinessWorkflow/OrderSettlement/BP_OrderSettlementBaseline`
- 函数：`EvaluateOrderBatch(Items) -> SettlementResult`

图中使用 `For Each Loop`、`Length`、`Break ST_OrderLine`、整数 `Multiply` 和 `Make ST_SettlementResult`。循环在第一次 Loop Body 执行时直接返回，`ConsumedLineCount` 则来自数组 `Length`；因此结果为 `2` 只证明读取了输入数组长度，不代表已经完整结算两条订单行。

| 输入 | 预期输出 |
|---|---|
| `(UnitPriceCents=5000, Quantity=2)`、`(7000, 3)` | `ConsumedLineCount=2`、`FirstLineTotalCents=10000`、`ProofMarker=24680` |

项目编译后，运行固定路径只读 Automation 测试：

```powershell
& "<UE_5.2>/Engine/Binaries/Win64/UnrealEditor-Cmd.exe" "<Project>/UEPluginDev.uproject" -unattended -nop4 -nosplash -NullRHI '-ExecCmds=Automation RunTests MCPBlueprint.BusinessWorkflow.OrderSettlementBaseline;Automation Quit' '-TestExit=Automation Test Queue Empty' -log
```

通过结果包含 `bPackagesCleanBefore=true`、`bPackagesCleanAfter=true` 和 `bExpectedOutput=true`。`SaveAsset` 会分别保存 Blueprint 与两个 User Defined Struct 包，不会执行 Save All。

![五节点固定订单结算基线](./Images/OrderSettlement/Baseline_5Node.png)

*真实 UE 5.2 Blueprint 图：已完成 MCP 创建、读回、编译、逐包显式保存、关闭编辑器、磁盘重载和固定输入执行。*

### V2 全行小计切片

`EvaluateOrderBatchV2` 保留上述 V1 基线不动，并完成了首个真实业务切片：遍历全部 `Items`，逐行计算 `UnitPriceCents * Quantity`，通过函数局部变量 `SubtotalCents` 的 `LocalGet` / `LocalSet` 累加，循环完成后才返回。`ConsumedLineCount` 仍来自数组 `Length`。

固定输入 `(5000, 2)`、`(7000, 3)`，并设置 `CustomerTier=0`、`Region=0`、`bFirstOrder=false`，磁盘重载后的运行结果为 `ConsumedLineCount=2`、`SubtotalCents=31000`。当前只验证全行小计；折扣、税、运费、积分、风控与最终决策尚未在此切片中实现。

```powershell
& "<UE_5.2>/Engine/Binaries/Win64/UnrealEditor-Cmd.exe" "<Project>/UEPluginDev.uproject" -unattended -nop4 -nosplash -NullRHI '-ExecCmds=Automation RunTests MCPBlueprint.BusinessWorkflow.OrderSettlementV2Subtotal' '-TestExit=Automation Test Queue Empty' -log
```

本阶段 `UEQuickStart.exe --test` 退出码为 0，`MCPBlueprint.Graph.PromotableOperators` 定向测试通过；订单结算共同前缀一次运行 3 个测试且全部通过。V2 日志包含 `MCPBP_ORDER_SETTLEMENT_V2_SUBTOTAL_RESULT=ConsumedLineCount=2 SubtotalCents=31000`，测试同时验证三个相关包在调用前后保持 clean。

![订单结算 V2 全行小计](./Images/OrderSettlement/V2_Subtotal_11Node.png)

*真实 UE 5.2 Blueprint 图：11 个节点，执行流为 Entry → For Each Loop → LocalSet，Completed → Return；数据流覆盖 Length、Break、Multiply、Add、LocalGet/LocalSet 与 V2 结果结构体。*

### 复杂订单策略验证

`BP_OrderSettlementComplex` 是从基线复制出的隔离示例，不修改上述 V1/V2 资产。它的 `EvaluateOrderBatchV2` 现为 64 节点、81 条边的单一连通长函数：遍历订单行累计小计，按 `CustomerTier` 选择折扣，叠加首单折扣，再按 `Region` 选择运费；`Region=1` 还会进入局部 Branch，首单免运费，非首单保持 1000。最后计算净额、8% 整数税费、总额、积分、风险分数与决策。长函数没有被拆成辅助函数，函数 ABI 保持不变。

长函数布局修复使用语义稳定键缓存、12 轮重心候选、真实几何交叉评分、有界相邻/全局交换与整层 Y 偏移搜索；每次布局的全局交换最多执行 4096 次候选评分，层偏移最多执行 2048 次指标评分。对上一版已保存布局运行 `FormatGraph(WholeGraph, Straight)`，真实读回为交叉线 `26 → 11`、`OverlapCount=0`、`BackwardEdgeCount=0`，平均 Pin 高差 `236 → 212`。重新生成全部 NodeGuid 的同构副本得到相同指标；布局自动化 17/17 通过。

固定订单行仍为 `(5000, 2)`、`(7000, 3)`，实际 `ProcessEvent` 运行覆盖五组路径：

| 用例 | Tier / Region / 首单 | 折扣 | 净额 | 税 | 运费 | 总额 | 积分 / 风险 / 决策 |
|---|---|---:|---:|---:|---:|---:|---|
| 普通订单 | `0 / 0 / false` | 0 | 31000 | 2480 | 800 | 34280 | `334 / 3 / 1` |
| 高等级首单 | `2 / 1 / true` | 2550 | 28450 | 2276 | 0 | 30726 | `307 / 3 / 1` |
| 高等级非首单 | `2 / 1 / false` | 1000 | 30000 | 2400 | 1000 | 33400 | `324 / 3 / 1` |
| Region 2 普通订单 | `0 / 2 / false` | 0 | 31000 | 2480 | 2000 | 35480 | `334 / 3 / 1` |
| 默认分支 | `99 / 99 / false` | 1500 | 29500 | 2360 | 2500 | 34360 | `318 / 3 / 2` |

`MCPBlueprint.BusinessWorkflow.OrderSettlementComplexPolicy` 在 UE 5.2 中返回 `Success`，五组输出均满足预期，相关包在调用前后保持 clean；订单结算前缀 4/4 回归通过。资产先显式保存，再关闭、重开、重新编译并截图；`UEQuickStart.exe --test` 同样退出码为 0。

以下截图是当前复杂业务逻辑的**验收基线**，使用 UE 原生 Comment Box 标出三个业务区；它们不是局部逻辑修改的 Before/After。总览图继续保留无框版本，避免远景缩放把节点简化为色块。后续真正修改某一段局部业务逻辑时，必须先保留修改前同画幅截图，再在节点、Pin 默认值或连线发生实际变化后拍摄修改后截图，并只框住该次变化范围。

![复杂订单策略全图](./Images/OrderSettlementComplex/ComplexPolicy_Full.png)

#### 当前局部基线：等级折扣与首单优惠

![当前基线：Comment Box 标注等级折扣与首单优惠](./Images/OrderSettlementComplex/ComplexPolicy_Discount.png)

#### 当前局部基线：地区运费与税费计算

![当前基线：Comment Box 标注地区运费策略](./Images/OrderSettlementComplex/ComplexPolicy_ShippingTax.png)

#### 当前局部基线：结果装配

![当前基线：保存重开后的 Comment Box 结果装配](./Images/OrderSettlementComplex/ComplexPolicy_ReopenedResult.png)

### 真实局部结构逻辑修改：Region 1 首单免运费

修改前，`Region=1` 的 Exec 直接进入 `Set ShippingCents(1000)`。修改后在原长函数内局部新增 `Get bFirstOrder`、`Branch` 和 `Set ShippingCents(0)`，并将 Case 1 重新连线为：首单走 True 免运费，非首单走 False 复用原 `1000` 节点，两路再汇入原 `Set NetCents` 下游。其他 Region Case、计税、总额、积分和决策连线保持不变。

| 路径 | 修改前 Shipping / Total / Points | 修改后 Shipping / Total / Points |
|---|---:|---:|
| `Tier=2, Region=1, bFirstOrder=true` | `1000 / 32590 / 325` | `0 / 31590 / 315` |
| `Tier=2, Region=1, bFirstOrder=false` | `1000 / 33400 / 334` | `1000 / 33400 / 334` |

Before 与 After 均为 2560×1440、1:1 缩放，并以未移动的 Region Switch 作为共同画幅锚点。After 中红色原生 Comment Box 只包围本次新增的 Getter、Branch、免运费 Set 以及被改线复用的 `1000` 节点。定向用例返回 `bExpectedOutput=true`，最终订单结算 4/4 回归均通过。

![结构逻辑修改前：Region 1 直接设置 1000 运费](./Images/OrderSettlementLogicRewire/Region1FirstOrder_Before.png)

![结构逻辑修改后：Region 1 增加首单免运费 Branch 与新连线](./Images/OrderSettlementLogicRewire/Region1FirstOrder_After.png)

### 多逻辑回归单元 1：首单折扣改为小计 5%

原图通过 `Get FirstOrderBonusCents` 把固定 750 加到等级折扣。本单元删除该 Getter，新增消费端 `Get SubtotalCents` 和整数 `Divide(B=20)`，将数据链改为 `Subtotal / 20 → Add Discount → Set DiscountCents`。首单用例变为 `Discount=2550, Net=28450, Tax=2276, Shipping=0, Total=30726, Points=307`，其他三组用例保持不变，完整回归 4/4 通过。

该场景曾暴露一个真实调用顺序限制：新建通配符 Divide 的 `PinDefaults` 早于连线类型提升执行，导致同一 patch 中的 `B=20` 被拒绝。当前已修复为：非 wildcard 默认值保持原顺序；新建节点上仍为 wildcard 的默认值会在全部 Connections 和 Unreal 原生类型提升后、布局与编译前，在同一事务内校验并应用。未解析 wildcard、非法默认值或连线失败都会整笔回滚；`PromotedType` 仍只用于 Real/Double 断言，不伪造 `int` 预提升。

![单元1修改前：固定首单奖励 Getter](./Images/MultiLogic/Unit1_FirstOrderPercent/Before.png)

![单元1修改后：小计 5% Divide 数据链](./Images/MultiLogic/Unit1_FirstOrderPercent/After.png)

### 多逻辑回归单元 2：未支持地区转人工复核

原图在 `Set TotalCents` 后始终执行 `Set Decision(1)`。本单元新增 `Get Region`、三个显式 Case 的 `Switch on Int` 和 `Set Decision(2)`；Region `0/1/2` 仍进入原批准节点，Default 进入人工复核，两路再汇入原 Return。测试表新增 Region 2 正向 Case，并将 Region 99 的 Decision 从 1 改为 2；五组业务输出和订单结算 4/4 自动化测试全部通过。

![单元2修改前：所有地区都直接批准](./Images/MultiLogic/Unit2_UnsupportedRegionReview/Before.png)

![单元2修改后：未知地区通过 Switch 进入人工复核](./Images/MultiLogic/Unit2_UnsupportedRegionReview/After.png)

### 多逻辑回归单元 3：积分基数排除运费

原积分链为 `TotalCents / 100 → PointsEarned`。本单元在原 Total Getter 与 Divide 之间新增消费端 `Get ShippingCents` 和整数 Subtract，改为 `(TotalCents - ShippingCents) / 100`；原 Points Divide 和 Make Struct 输入继续复用。五组积分依次为 `334 / 307 / 324 / 334 / 318`，金额、风险和决策字段保持不变，完整回归 4/4 通过。

![单元3修改前：总额直接进入积分 Divide](./Images/MultiLogic/Unit3_PointsExcludeShipping/Before.png)

![单元3修改后：总额先减运费再计算积分](./Images/MultiLogic/Unit3_PointsExcludeShipping/After.png)

## 安全与行为边界

- 新建、移动或复制蓝图时，目标必须是 `/Game` 下合法的独立包路径；若内存或磁盘已有同名包会拒绝操作。
- 关卡蓝图可以通过对象路径读取和编辑图表，但 `SaveAsset`、`DeleteAsset`、`RenameAsset` 与 `DuplicateAsset` 不处理关卡蓝图，因为这些操作会影响所属关卡包；请通过 World Editor 工作流处理关卡本身。
- `CloseAsset` 会返回之前是否打开，并核验蓝图及其子对象的所有编辑器是否真正关闭；用户取消关闭时返回错误。
- `ReloadBlueprintFromDisk` 默认只做只读 dry-run，返回 Package Dirty、编辑器打开状态，以及有界的磁盘路径、大小、时间和 MD5 元数据。由于 Unreal 会重载整个 Package，工具会稳定枚举顶层 `RF_Public|RF_Standalone` 资产；若除目标 Blueprint 外还有任何资产，dry-run 与正式执行都会 fail-closed，并用有界冲突列表和 `bCanApply=false` 结构化报告。正式执行必须同时传入 `bDryRun=false`、`bDiscardUnsavedChanges=true` 与 `bAcknowledgeUndoHistoryReset=true`。工具会关闭并核验所有顶层 Package 资产的编辑器，调用 Unreal 官方包重载 API，按原对象路径重新取得替换后的 Blueprint，验证新 Package 为 clean，并可选择重开编辑器。一旦 Unreal 已报告真实重载，所有后验验证错误只会在完成请求的重开尝试后统一汇总，`bReloadedFromDisk` 仍保持 true。为防止事务历史保留已被替换的对象引用，Unreal 在包重载时必然重置整个编辑器的 Undo/Redo 历史；工具会在执行前后明确报告该影响。工具不会保存资产、执行 Save All、绕过包重载回调或手工清理引用。
- 成员变量和局部变量的默认值会在修改前按 Unreal K2 Pin 规则校验；不合法的 UE 文本值会被拒绝。
- `ModifyVariable` 可选接收成员变量的 `Category` 与 `Tooltip`。空 `Category` 恢复 Unreal 默认类别；空 `Tooltip` 删除该元数据。该工具会在事务中更新变量声明，只把蓝图标记为已修改而不强制结构重建，支持 Undo，编译失败会回滚，且绝不自动保存；`GetBlueprintOverview` 会在存在时返回 `Tooltip`。
- 存在未加载派生蓝图时，变量类型变更、重命名和删除会被拒绝。请先加载派生蓝图以检查继承引用；删除声明前必须先移除子蓝图引用。
- 启用自动编译时，事务型蓝图修改会返回有界编译诊断，并在终态不是 `UpToDate` 或 `Warning` 时回滚且明确报告回滚状态；禁用时结果会返回 `CompileStatus: Skipped`。
- 组件名称会对完整蓝图成员命名空间进行校验。短组件类名存在歧义时会被拒绝；请使用完整类路径消除歧义。
- 组件类必须允许由蓝图创建。未显式指定父级的新 SceneComponent 会挂到唯一的本地、继承或原生场景根；存在多个根时会拒绝操作，不会创建第二个根。
- SCS 层级编辑会为 Undo 快照本地层级，并在编译失败时回滚。`SetRootComponent` 只替换唯一且明确的本地场景根，不会替换继承或原生根。
- `GetComponentProperties` 使用 `Offset`/`MaxResults` 分页（每页最多 100 个属性），单个导出值最多返回 8,192 个字符，并应用整体序列化属性预算与续页元数据。
- `GetClassDefaults` 使用与 `SetClassDefaults` 完全一致的可编辑属性筛选，返回稳定分页及声明类/继承元数据，并同时限制单值为 8,192 个字符和整体序列化属性预算。
- Class Default 与组件模板属性映射会省略原生固定数组反射属性（`ArrayDim > 1`），因为当前映射协议无法安全寻址固定数组的单个元素；普通 `TArray` 属性仍可通过 UE 文本格式读写。
- `GetBlueprintOverview` 对每个分区应用稳定的 `Offset`/`MaxResults` 分页（每个分区最多请求 50 项），同时限制各分区序列化预算；变量默认值最多返回 8,192 个字符，每个函数最多返回 16 个输入和 16 个输出签名 Pin。
- `ListBlueprintMembers` 对函数、Custom Event、Dispatcher 与局部变量使用统一稳定分页；每个语义方向最多返回 16 个签名 Pin，并区分 `SignatureDirection` 与 Entry/Result 节点使用的反向物理 `NodeDirection`。
- `GetComponents` 返回稳定分页的本地扁平列表和继承列表（各最多 100 项），并保留最多 200 个节点、深度最多 64 的本地层级树。扁平遍历使用显式栈，超过 10,000 个组件或深度 256 时报告截断。
- `GetGraphDetail` 单次最多请求 25 个稳定排序的节点，并受序列化页预算限制；每个节点的可见 Pin 独立分页（最多 64 个）且受单节点 Pin 预算限制，每个 Pin 最多返回 8 个有界连接，并提供明确的截断与续页元数据。
- Asset Registry 仍在扫描时会拒绝列举和删除蓝图，避免分页与引用检查建立在不完整数据上。
- 删除资产不可 Undo。未传 `bForce` 时会拒绝删除被引用资产，并使用编辑器常规删除流程保留实时引用；强制删除可能清空引用并破坏依赖资产。
- 若目标资产仍被当前会话的 Undo/Redo 事务历史持有，`DeleteAsset` 可能返回 `locked or in use`。不要改用 `bForce` 掩盖状态；先保存需要保留的工作并重启编辑器，再以 `bForce=false` 重试。
- 截图文件输出限定在 `Saved/MCPBlueprint/Screenshots`；绝对路径、路径穿越和已存在的链接/重解析点路径会被拒绝。除非显式传入 `bOverwrite=true`，否则不会覆盖已有 PNG；这些检查用于降低链接路径与误覆盖风险，但不宣称消除外部文件系统竞争。
- 仅源码静态兼容审查不能代替在各目标引擎版本中的真实编译与测试。

- 工具 Schema 会拒绝未知顶层参数，并在执行前递归校验图补丁、函数/事件签名、Pin 默认值、组件属性与 Class Defaults 的结构。
- HTTP 请求体在引擎 HTTP 服务器完成接收后按 1 MiB 上限拒绝；序列化响应上限为 16 MiB。工具文本最多 1 Mi 字符；图片最多 4 张、单张最多 8 Mi Base64 字符、合计最多 12 Mi。
- 编译诊断最多返回 200 条错误/警告；单条最多 4,096 字符、合计最多 256 Ki 字符，每条最多 32 个去重 NodeGuid。`TotalErrors`、`TotalWarnings`、实际返回数量与 `bTruncated` 会明确报告省略情况。`CompileBlueprint` 和 `GetCompileErrors` 在非成功终态返回 MCP 错误；可用 `bWarningsAsErrors` 把警告也视为失败。
- 请求超时只限制 Dispatcher 等待时间，无法抢占已在游戏线程同步执行的 Blueprint/UObject 操作。客户端超时后应先确认编辑器状态，不要立即重试写操作。

## 配置

打开 **Project Settings > Plugins > MCP Blueprint**。

| 设置 | 默认值 | 说明 |
|---|---:|---|
| Port | 8766 | 首个尝试的 HTTP 端口。 |
| Auto Start | true | 编辑器模块加载时启动服务器。 |
| Request Timeout | 30 秒 | 普通工具超时。 |
| Compile Timeout | 120 秒 | 编译/图补丁工具超时。 |
| Auto Compile After Modify | true | 支持的写操作后自动编译。 |

## 当前测试重点

当前源码的 UE 5.2–5.8 Win64 BuildPlugin 矩阵均已通过，七个引擎版本都已部署对应兼容包且 DLL 哈希与包一致。UE 5.2、5.3、5.5–5.8 的 initialize、`tools/list`（53）和 ping 均通过；UE 5.4 已确认 EnginePlugin 加载、注册 53 个工具并启动 endpoint。这些兼容包不是最终 Fab 提交产物：描述符仍保持 Beta，正式发布元数据与产品图标需要取得真实 listing UUID 后重新打包。

## 已验证截图

[`Images/Fab`](./Images/Fab/) 中的 PNG 均由真实 Unreal Blueprint Graph 控件直接捕获，未合成伪 UI，也不含本机文件路径。各图说明与对应工作流见 [FAB_LISTING.md](./FAB_LISTING.md)。这些图片不会被描述为 HTTP 响应或变量 Details 面板截图。算术节点使用两张真实画面共同证明持久化后的 Real/Double `Add → Subtract → Multiply → Divide → Result` 链：全图缩放图覆盖左侧与中段，节点聚焦图覆盖 `Subtract → Multiply → Divide → Result`；原因是 UE 5.2 的离屏 Graph Widget 全图捕获有时不绘制最右侧节点本体。

当前 55 工具源码已通过 UE 5.2 编译。`MCPBlueprint.Function.ModifySignatureSafety` 已在 NullRHI 与真实非 NullRHI 编辑器中通过，使用已落盘声明 Blueprint 与外部 Caller，覆盖 dry-run 零修改、显式批准、AddInput/AddOutput、保持 PinId/连线的重命名、数据风险门禁删除/改类型、一次 Undo、注入失败回滚，以及通过 UE 官方同批重载声明/Caller Package、按对象路径重新取得资产并验证引用修复后的持久化状态。`MCPBlueprint.Asset.ReloadBlueprintFromDiskSafety` 仍保留其独立的非 NullRHI 安全覆盖。此前 `RenameFunction`、Struct/Enum、图表与 HTTP 回归对其测试提交仍然有效。当前文档中的 UE 5.2–5.8 打包/部署矩阵是 53 工具的历史结果；刷新矩阵前不会宣称 55 工具已完成跨版本等价验证。

打包验证证明各目标引擎可以编译插件并获得预期目录结构，但不能替代每个引擎版本内对全部 MCP 操作的运行时验证。其余工具的跨版本真实编辑器回归、保存/重启持久化与清理行为仍会继续推进。

## 已知限制

- `Auto` 局部让位被刻意限制为最多 128 个既有节点、128 轮闭包扩展和 100000 Graph Units。触及 Function Entry 或 Comment 包围边界时会 fail-closed，而不会扩张成不安全的整图重排；此时应缩小插入块或显式整理受保护区域。
- 同一会话中经过大量事务修改的资产可能仍被 Undo/Redo 历史持有，导致非强制删除失败；处理方式见上方 `DeleteAsset` 安全边界。后续计划提供不清空无关 Undo 历史的定向处理与更明确诊断。

## 后期路线图

开发已恢复，并继续按可独立验证的有界单元推进。当前描述性变量元数据单元覆盖 `Category` 与 `Tooltip`；以下条目是剩余方向，不代表具体版本或日期承诺。

- **事件签名与重构：** 把已验证的函数签名安全模型扩展到 Custom Event 和 Event Dispatcher 签名与标志。
- **剩余变量元数据：** 扩展变量编辑，覆盖访问控制、Expose on Spawn、SaveGame、复制和 RepNotify 等设置。
- **图表生命周期与影响分析：** 创建、重命名和删除受支持的图表类型；移除已实现接口；重构前查询引用与受影响资产。
- **蓝图类型：** 创建 Blueprint Interface、Function Library、Macro Library 等其他蓝图资产类型。
- **编辑体验与诊断：** 改进成员文档和面向调试的状态检查能力。

以上内容是候选方向，不承诺具体版本或完成时间；后续将根据本轮测试与真实使用证据继续调整。

## AI 辅助使用

如果对某个工具、参数或操作流程不清楚，可以把本文档提供给 AI，让 AI 先阅读文档，再根据当前 Unreal Editor 会话中实际可用的 MCP 工具尝试操作。涉及写入时请先确认目标资产，并在操作后重新读取受影响的蓝图状态，不要只凭工具返回成功就认定操作完成。

## 支持

如有问题、反馈或希望加入 UE 技术交流群，请访问以下统一联系方式页面。

- **联系入口：** [邮箱、微信群和 X](https://mengzhishanghun.github.io/mengzhishanghun/contact/)
