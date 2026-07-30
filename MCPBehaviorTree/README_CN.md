[English](./README.md) | [中文](./README_CN.md)

# MCP Behavior Tree

MCP Behavior Tree 是一个自包含的 Unreal Editor 插件，可通过 Model Context Protocol 创建、查看、编辑、验证并保存行为树与黑板资产。插件支持 Unreal Engine 5.2 及更高版本，不依赖其他用户插件。

## 连接

插件会在 Unreal Editor 中启动 MCP HTTP 服务。默认端口为 `8767`；若端口被占用，应以插件运行时报告的实际端口为准，不要假定始终使用默认端口。

资产参数使用 `/Game/AI/BT_Enemy`、`/Game/AI/BB_Enemy` 这类 Unreal 内容路径。新建或编辑后的资产保持脏状态，直到 `SaveAsset` 成功。

`SaveAsset`、`OpenAsset` 和 `CloseAsset` 只接受行为树或黑板资产。编辑器图无法同步或 Unreal 无法写入包时，保存会直接失败；因此 MCP 返回成功即表示包保存成功。

## 工具

| 领域 | 工具 |
|---|---|
| 资产 | `CreateBehaviorTree`、`CreateBlackboard`、`SetBehaviorTreeBlackboard`、`ListBehaviorTrees`、`GetBehaviorTreeDetail`、`SaveAsset`、`OpenAsset`、`CloseAsset` |
| 黑板 | `AddBlackboardKey`、`RemoveBlackboardKey` |
| 树结构 | `AddCompositeNode`、`AddTaskNode`、`ConnectNodes`、`RemoveNode` |
| 节点 | `SetNodeProperties`、`AddDecorator`、`AddService` |
| 验证 | `ValidateBehaviorTree` |

`GetBehaviorTreeDetail` 会返回编辑工具可复用的稳定 `NodeId`、关联黑板及键、嵌套树结构、装饰器、服务、反射导出的可编辑属性和游离节点。大结果受深度、节点数、属性数和文本上限保护，未完整返回时会设置 `bTruncated`。`ListBehaviorTrees` 支持确定性排序、过滤以及 `Offset`/`MaxResults` 分页。

## 推荐流程

1. 使用 `CreateBlackboard` 创建黑板。
2. 使用 `AddBlackboardKey` 添加键。
3. 使用 `CreateBehaviorTree` 创建行为树并传入黑板路径。
4. 先在 `Root` 下添加 Composite，再添加任务、装饰器和服务。
5. 使用 `GetBehaviorTreeDetail` 获取节点 ID 和可编辑属性。
6. 运行 `ValidateBehaviorTree`。
7. 对每个脏的行为树或黑板资产调用 `SaveAsset`。

## 黑板键规则

支持的键类型为 `Bool`、`Int`、`Float`、`Vector`、`Rotator`、`Name`、`String`、`Object`、`Class` 和 `Enum`。

- `Object` 与 `Class` 键可指定 `BaseClass`。
- `Enum` 键必须提供 `Enum`；无关类型若传入 `BaseClass` 或 `Enum` 会被拒绝。
- `KeyName` 不得为空或 `None`。
- 键选择器按 Unreal 原生过滤语义校验，包括 Object/Class 基类约束和 Enum 身份；同名但类型不兼容的键仍然无效。
- `RemoveBlackboardKey` 会扫描项目中的行为树；存在引用或无法完成扫描时默认拒绝删除，只有显式设置 `bForce=true` 才接受风险。

## 编辑行为

编辑工具使用 Unreal 事务，并将受影响的包标记为脏。节点属性采用 Unreal import-text 字符串：布尔值使用 `true`/`false`，枚举使用名称，向量使用 `(X=0.0,Y=0.0,Z=0.0)`。黑板选择器属性接收黑板键名，并按选择器的原生类型过滤器验证。

`SimpleParallel` 具有 `Default` 与 `Background` 输出引脚。添加或移动后台子树时使用 `ParentOutputPin="Background"`。保存前应运行验证；验证会报告结构错误、缺失类、无效键绑定、无效装饰器中止模式和未连接节点。

## 与 Unreal Engine 5.8 官方能力比较

Unreal Engine 5.8 的 Epic `AIModuleToolset` 提供 7 个只读行为树辅助能力：获取黑板、根级装饰器、节点列表、单个/全部节点深度、子节点和引用的子树。MCP Behavior Tree 通过 `GetBehaviorTreeDetail` 覆盖黑板、装饰器、层级、子节点和深度语义，并额外提供创建与编辑能力。

Epic 工具集属于 Experimental，并绑定 UE 5.8 的 Toolset/MCP 基础设施。为了保持 UE 5.2+ 兼容和插件自包含，本插件不能依赖它。若某项官方专属行为未被本文档列出的插件工具覆盖，应在 UE 5.8 中调用 Epic 工具集，而不能假定早期引擎也具备该能力。

## 限制

- 插件仅用于编辑器，不控制运行时或打包后的游戏。
- 本插件不编写 Blueprint 任务、装饰器和服务的内部逻辑。请先通过具备 Blueprint 能力的工具创建并编译这些类，再传入生成的 `_C` 类路径。
- Behavior Tree 编辑器的私有图 API 会随引擎版本变化。插件包含 UE 5.2+ 兼容路径，但仍应在目标引擎版本中验证资产。
- 若行为树已有运行时节点却缺失编辑器图，插件会拒绝把它重建为空图。请先在 Behavior Tree 编辑器中修复资产，再使用 MCP 编辑或保存工具。
- 源码发生变化不代表既有插件包已经重新打包或发布。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
