# Unreal Engine MCP 插件套件总览与路线图

> 状态快照：2026-08-10
>
> 统计口径：以各插件当前源码的模块注册表为准，不把测试辅助类计入工具数。
>
> 路线图用于说明职责边界和候选方向，不承诺具体版本或完成时间。

## 状态标记

| 标记 | 含义 |
|---|---|
| ✅ 已实现 | 已在当前源码注册，可通过 MCP 工具调用。 |
| 🧪 验证中 | 工具已经实现，但仍需补充真实编辑器、拒绝路径、保存重启或跨版本回归。 |
| 🗓 已规划 | 已确认归属和目标，但尚未进入当前工具集。 |
| 💡 候选 | 有价值但尚未确定版本、范围或最终插件归属。 |

“已实现”只表示当前源码具备该能力，不等于所有输入组合、引擎版本和资产状态都已经完成运行验证。

## 套件设计原则

1. 按 Unreal Engine 资产和编辑领域划分插件，不按调用工具或文件格式划分。
2. 领域专属能力放入最具体的插件；项目级维护、迁移和构建能力不复制到各领域插件。
3. 写操作默认先读取和预检，危险操作提供 `bDryRun` 或显式风险确认，并在写入后读回、编译或保存验证。
4. 各插件保持独立部署。跨领域工作流由 AI 组合多个 MCP 服务，不建立不必要的源码依赖。
5. 当前阶段优先完成已有工具的真实编辑器完整回归，再扩展新的大类能力。

## 当前插件一览

| 插件 | 当前源码工具数 | 核心职责 | 当前状态 |
|---|---:|---|---|
| MCPAnimation | 29 | 动画蓝图、状态机、Skeleton、AnimSequence、Montage、Notify 与 BlendSpace | ✅ 已实现；部分集合分页和完整运行回归仍需补充 |
| MCPBehaviorTree | 18 | Behavior Tree、Blackboard、节点、装饰器、服务与结构验证 | ✅ 已实现；不负责 Blueprint Task 等类内部逻辑 |
| MCPBlueprint | 46 | Blueprint 结构、成员、图表、组件、父类、编译、资产生命周期与截图 | ✅ 已实现；现有工具完整回归是当前重点 |
| MCPLevelSequence | 23 | Sequence、Binding、Track、Section、关键帧、Camera Cut 与视图截图 | ✅ 已实现；完整构建和运行回归仍需补充 |
| MCPMaterial | 17 | Material/Material Instance、表达式图、参数、编译与预览 | ✅ 已实现；高级材质结构仍有限制 |
| MCPNiagara | 23 | System、Emitter、模块栈、参数、Renderer、编译与预览 | ✅ 已实现；真实编辑器和跨版本验证仍需补充 |
| MCPUMG | 26 | Widget Blueprint、Widget Tree、属性、Slot、动画、事件与绑定 | ✅ 已实现；高级事件图操作可组合 MCPBlueprint |
| MCPWorldEditor | 71 | 关卡、Actor、组件、视口、World Partition、流送、地形与植被 | ✅ 已实现；继续按真实场景补充工具和回归 |

当前 8 个插件合计注册 **253 个 MCP 工具**。

## MCPAnimation

### ✅ 当前已实现

- Animation Blueprint 与状态机发现、详情读取、状态和 Transition 编辑。
- Skeleton 与 AnimSequence 列表和有界详情读取。
- Montage 创建、Section 管理与详情读取。
- Notify 类发现、Notify 添加和删除。
- BlendSpace 列表、详情、创建和 Sample 编辑。
- Animation Blueprint 创建、编译、保存和打开。

### 🗓 后续方向

- 为当前只有有界输出的详情集合增加稳定分页。
- 根据真实任务补充动画曲线和压缩数据能力。
- 完成保存、重开、Undo/Redo 及支持引擎版本的统一运行回归。

## MCPBehaviorTree

### ✅ 当前已实现

- Behavior Tree 与 Blackboard 创建、列表和详情读取。
- Blackboard 绑定、Key 添加和删除。
- Composite、Task、Decorator、Service 节点添加与属性编辑。
- 节点连接、删除和完整结构验证。
- 资产保存、打开和关闭。

### 🗓 后续方向

- 继续完善异常资产、缺失编辑器图和复杂 Decorator 组合的诊断。
- Blueprint Task、Decorator、Service 的内部 Blueprint 逻辑继续由 MCPBlueprint 负责，不在本插件重复实现。
- 补全复杂树结构、Abort Mode、Simple Parallel 和保存重启回归。

## MCPBlueprint

### ✅ 当前已实现

- Blueprint 列表、结构总览、统一成员列表和 Graph 详情。
- 变量、函数、Custom Event、局部变量、接口和 Event Dispatcher 管理。
- Graph Node 搜索、声明式 Graph Patch、Pin 默认值和可选 Pin 编辑。
- 确定性图表布局、Comment Box 和 Graph Screenshot。
- Component 添加、属性编辑、重命名、删除、复制、层级和 Root 管理。
- Class Defaults 读取与修改。
- Blueprint 创建、父类修改、编译、保存、打开、关闭、删除、重命名和复制。
- 父类修改的 Dry Run、数据丢失确认、继承循环拒绝和编译失败回滚。

### 🗓 已确认的下一项能力

#### User Defined Struct

- 创建、读取和修改 `UUserDefinedStruct`。
- 添加、重命名、删除字段以及修改字段类型和默认值。
- 字段重命名应保留稳定 GUID；删除或改类型前必须报告潜在数据丢失。
- 拒绝直接或间接的结构体循环引用。
- 修改后刷新并编译引用它的 Blueprint，报告受影响的 DataTable 和 DataAsset。

#### User Defined Enum

- 创建、读取和修改 `UUserDefinedEnum`。
- 添加、重命名、删除和调整枚举项。
- 删除或重排前扫描引用并报告已有序列化值可能发生的语义变化。
- 修改后刷新并编译引用它的 Blueprint。

### 💡 后期候选

- 成员签名重构、变量元数据与更完整的影响分析。
- Function Graph、Macro Graph 等图表生命周期管理。
- Interface、Function Library、Macro Library 等 Blueprint 类型的完整专用流程。
- 对大量事务、Undo 持有和非强制删除失败提供更明确的定向诊断。

### 职责边界

- 编辑器资产 `UUserDefinedStruct`、`UUserDefinedEnum` 属于 MCPBlueprint。
- C++ `USTRUCT`、`UENUM` 的源码生成、重构和编译属于未来的 MCPDeveloperTools。
- 通用资产 Redirector 修复和项目级 CoreRedirects 不属于 MCPBlueprint。

## MCPLevelSequence

### ✅ 当前已实现

- Level Sequence 列表、详情、创建、Playback Range 和帧率设置。
- Binding 添加、列表和删除。
- Track 类型发现、Track/Section 添加删除和范围设置。
- Keyframe 读取、写入和删除。
- Camera Cut 添加和编辑。
- 保存、打开、关闭和 Sequencer View 截图。

### 🗓 后续方向

- Subsequence 写入和 Spawnable 管理。
- Event Payload、更多 Track/Class 和 Channel 类型。
- Folder、Tag 等 Sequencer 组织能力。
- 完成资产保存、关闭交互、Undo/Redo 和跨版本运行回归。

## MCPMaterial

### ✅ 当前已实现

- Material/Material Instance 列表、详情和参数读取。
- Material 与 Material Instance 创建。
- Material 属性、实例参数和表达式图编辑。
- 表达式搜索、Graph Patch、断开连接和确定性布局。
- 编译、保存、打开、关闭和 Preview Capture。

### 🗓 后续方向

- Material Layer 与 Blend 的结构化参数编辑。
- 当前通用表达式图补丁无法完整表达的特殊节点。
- 更多高级 Material Instance 参数类型和资源引用写入。
- 使用真实复杂材质完成编译、保存重启与跨版本回归。

## MCPNiagara

### ✅ 当前已实现

- Niagara System/Emitter 列表和详情读取。
- System 创建、Emitter 添加、删除、重命名和属性编辑。
- Module 搜索、添加、删除、排序、详情和参数编辑。
- System Parameter 与 User Parameter 管理。
- Renderer 类型和属性编辑。
- 编译、保存、打开和 Preview Capture。

### 🗓 后续方向

- Object 和 Data Interface 类型参数的安全写入。
- User Parameter 重命名及引用迁移。
- 在不依赖引擎 Private API 的前提下改善 Emitter Merge Cache 相关诊断。
- 完成真实编辑器、保存重启和跨版本运行验证。

## MCPUMG

### ✅ 当前已实现

- Widget Blueprint 列表、Widget Tree、属性和 Slot 读取。
- Widget 类搜索、添加、删除、重命名、换父级、复制和包裹。
- Widget/Slot 属性写入。
- UMG Animation 创建、删除、Track、Key 和详情管理。
- Widget Event 绑定、解绑和动作添加。
- Widget Blueprint 创建、Blueprint Settings 和 Property Binding。
- Widget Screenshot。

### 🗓 后续方向

- 继续补充复杂 Panel Slot、动画 Channel 和绑定场景的真实回归。
- 高级 Blueprint Event Graph 逻辑继续组合 MCPBlueprint，不在 MCPUMG 复制图表编辑器。
- 根据真实任务补充目前 Widget 属性反射无法安全覆盖的专用控件操作。

## MCPWorldEditor

### ✅ 当前已实现

- Level 创建、打开、保存、详情和 Level Blueprint 获取。
- Actor 搜索、生成、复制、删除、重命名、附加、分离和分组。
- Component 添加、删除以及 Actor 属性、Transform、Tag、可见性、锁定和 Collision 编辑。
- Selection、Viewport Camera、Viewport Type、Render Mode、截图和 Focus。
- World Settings、Light、Material、Static Mesh 和 Material Instance 操作。
- Undo/Redo、受限 Console Command、Line Trace、PIE 和 Editor State。
- World Partition、Data Layer、Region Loader、Streaming Level 和 Actor 跨关卡移动。
- Landscape、Foliage、Actor Folder 和 Actor Layer。

### 🗓 后续方向

- 继续以真实关卡工作流暴露的缺口为依据扩展，不预先批量复制编辑器菜单。
- 完成所有不可完全撤销操作、Selection 恢复和跨关卡引用场景的持续回归。
- 与其他领域插件组合验证关卡内 Blueprint、Material、Niagara、Sequence 和 UMG 工作流。

## 计划中的项目级插件

下面两个名称是当前建议，最终名称可在正式开发前确认。

### 🗓 MCPProjectTools

负责通用资产和项目内容维护，不绑定某一种资产类型：

- `ScanAssetRedirectors`：扫描 Redirector、目标资产和引用范围。
- `FixupAssetRedirectors`：限定目录、支持 Dry Run，并在引用资产成功保存后修复 Redirector。
- `ScanBrokenReferences`：扫描失效的软引用、对象路径和包路径。
- `ResaveAffectedAssets`：对明确范围内的迁移资产进行有界 Resave，并返回逐项结果。
- 项目内容迁移前后的一致性检查。

### 🗓 MCPDeveloperTools

负责源码、配置、构建和类型迁移等开发工具链能力：

- `PlanCoreRedirects`：生成 Class、Struct、Enum、Function、Property 或 Package Redirect 建议。
- `ApplyCoreRedirects`：显式授权后修改指定配置，检查重复、冲突和循环。
- C++ `UCLASS`、`USTRUCT`、`UENUM` 的受控创建和重构。
- 项目编译测试、日志读取与问题定位能力；具体实现必须遵守项目已有构建入口。
- CoreRedirects 生效、资产 Resave 和兼容条目保留周期的验证。

## 暂不单独建立的插件

### MCPSchema

当前不为 Struct/Enum 单独建立 MCPSchema。只有未来同时出现 DataTable Schema、Gameplay Tags、Data Registry、Primary Asset 类型和其他项目数据模型需求，并且 MCPBlueprint 的职责明显过载时，再评估提取。

### MCPDataAsset

DataAsset 的结构定义、数据编辑和运行时查询职责差异较大。在形成真实、重复的需求前，不创建覆盖所有数据资产的通用写入插件，也不把 DataTable 行数据编辑塞入 MCPBlueprint。

## 建议实施顺序

1. **现有能力完整回归**：优先测试 8 个插件已经注册的工具，包括正常路径、拒绝路径、事务回滚、分页与输出上限、保存重启和清理。
2. **MCPBlueprint Struct/Enum**：实现并验证 User Defined Struct 与 User Defined Enum 的创建、读取和安全修改。
3. **MCPProjectTools Redirector**：先实现扫描和 Dry Run，再实现有界 Fixup 与 Resave。
4. **MCPDeveloperTools CoreRedirects**：先做只读规划和冲突检测，再考虑配置写入与构建验证。
5. **按真实任务补齐领域缺口**：只根据已经出现并可复现的场景扩展各专业插件。

## 完成标准

新能力只有同时满足以下条件，才能从“已规划”标记为“已实现并验证”：

1. 工具已注册，输入 Schema 与运行时验证一致。
2. 正常路径、无效输入、冲突输入和权限/只读路径均有验证。
3. 写入后完成读回，并根据资产类型执行编译、保存或重开验证。
4. 失败不会留下未报告的部分状态；无法完全回滚时明确返回状态可能已变化。
5. 工具说明、中英文用户文档和实际行为一致。
6. 临时测试资产和生成文件已按范围清理，不污染用户项目与版本控制。
