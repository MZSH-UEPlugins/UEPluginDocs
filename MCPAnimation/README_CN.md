[English](./README.md) | [中文](./README_CN.md)

# MCPAnimation

MCPAnimation 是一个自包含的 Unreal Editor 插件，通过 HTTP MCP 向 AI 客户端提供动画蓝图、状态机、Skeleton、AnimSequence、Montage、AnimNotify 与 BlendSpace 的发现、读取和编辑能力。插件支持 Unreal Engine 5.2 及以上版本，不依赖其他用户制作的插件。

## 连接方式

默认端点为 `http://127.0.0.1:8760/mcp`。若端口被占用，服务器会继续尝试后续端口；请以编辑器中显示的实际端点为准。服务器实现 `initialize`、`tools/list`、`tools/call` 和 `ping`。

## 工具（29 个）

### 发现与只读详情

| 工具 | 用途 |
|---|---|
| `ListAnimBlueprints` | 稳定分页列出 Animation Blueprint。 |
| `GetAnimBPOverview` | 读取 Animation Blueprint、AnimGraph 和状态机概览。 |
| `GetStateMachineDetail` | 有界读取状态、过渡和规则图详情。 |
| `ListSkeletons` / `GetSkeletonDetail` | 发现 Skeleton，并读取骨骼、虚拟骨骼、兼容骨骼、Blend Profile 与 Slot Group。 |
| `ListAnimSequences` / `GetAnimSequenceDetail` | 发现 AnimSequence，并读取时长、采样率、Notify、float curve 和压缩状态。 |
| `ListMontages` / `GetMontageDetail` | 发现 Montage，并读取 Section、Slot、片段、Notify 与混合设置。 |
| `ListNotifyClasses` | 发现原生及 Blueprint AnimNotify/AnimNotifyState 类。 |
| `ListBlendSpaces` / `GetBlendSpaceDetail` | 发现 BlendSpace，并读取轴与采样点。 |

### Animation Blueprint 与状态机编辑

| 工具 | 用途 |
|---|---|
| `CreateAnimBlueprint` | 为指定 Skeleton 创建 Animation Blueprint。 |
| `AddState` / `RemoveState` | 添加或移除状态。 |
| `SetStateAnimation` | 仅在能确认唯一直接驱动状态结果的 Sequence Player 时设置动画。 |
| `AddTransition` | 在两个状态之间创建过渡。 |
| `SetTransitionRule` | 设置受支持的过渡规则。 |
| `CompileAnimBlueprint` | 编译并返回包括 AnimGraph、状态图和过渡图在内的诊断。 |

### Montage、Notify 与 BlendSpace 编辑

| 工具 | 用途 |
|---|---|
| `CreateMontage` | 从 AnimSequence 创建 Montage。 |
| `AddMontageSection` / `SetMontageSection` / `RemoveMontageSection` | 管理 Montage Section。 |
| `AddNotify` / `RemoveNotify` | 在 AnimSequence 或 Montage 上添加、移除 Notify。 |
| `CreateBlendSpace` | 在轴范围有效时创建 BlendSpace。 |
| `SetBlendSpaceSamples` | 事务化替换最多 256 个已验证的采样点；失败时回滚。 |

### 资产生命周期

| 工具 | 用途 |
|---|---|
| `SaveAsset` | 显式保存已修改资产。 |
| `OpenAsset` | 在 Unreal Editor 中打开资产。 |

## 路径、事务与保存

- 先调用对应的 `List*` 工具取得规范资产路径。资产写操作仅允许有效的 `/Game` 项目内容路径。
- 写操作在编辑器游戏线程执行，并在适用时使用 Undo 事务；成功后把资产标记为 Dirty。
- 修改不会隐式保存到磁盘。确认结果后调用 `SaveAsset`；如需撤销，可使用 Unreal Editor 的 Undo。
- 编辑器处于 PIE、保存或垃圾回收等忙碌状态时，写工具会拒绝执行并返回原因。

## Schema、分页与有界输出

- 输入使用严格 JSON Schema；未知字段会被拒绝，数组元素也会递归校验。
- 列表工具使用 `Offset` 和 `MaxResults`，每页最多 256 项，并返回 `TotalCount`、`Returned`、`HasMore` 和可用时的 `NextOffset`。
- 详情工具对骨骼、状态、Notify、曲线等集合设置 256 项上限，并返回真实总数与 `*Truncated` 标记。详情集合当前不支持翻页；超过上限时，请在 Unreal Editor 中检查完整数据，或等待后续分页能力。

## 与 UE 5.8 官方动画 MCP 的能力边界

UE 5.8 官方 `AnimationAssistantToolset` 的重点是 Level Sequence、Control Rig、Sequencer 通道和 Sequencer 曲线。MCPAnimation 不复制这些跨域工具；本插件聚焦 Animation Blueprint、Skeleton、AnimSequence、Montage、Notify 和 BlendSpace。需要 Sequencer 或 Control Rig 工作流时，应使用对应的 Level Sequence MCP 能力。

`GetAnimSequenceDetail` 只读返回 float curve 摘要、骨骼/曲线压缩设置、压缩有效性和近似大小。当前不提供曲线或压缩写入：这些操作可能重建动画数据模型、触发耗时重压缩，并且 UE 5.2–5.8 的编辑 API 存在演进；在无法同时保证事务回滚、版本兼容和有界执行前，自动写入的资产损坏风险高于收益。

## 限制

- 本插件是 Editor-only，不提供运行时或网络游戏复制能力。
- 详情接口提供结构化摘要，不导出所有曲线键值或逐骨骼动画轨道。
- 复杂或歧义的状态结果图不会被猜测修改；请先调整为单一直接驱动结构或使用 Unreal Editor 手工编辑。
- 工具不会执行 C++ 编译、Unreal 项目构建或插件打包。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
