[English](./README.md) | [中文](./README_CN.md)

# MCP Niagara

MCP Niagara 在 Unreal Editor 中提供面向 Niagara System、Emitter、模块栈、参数、Renderer、编译状态和资产生命周期的 MCP 工具。

## 资产发现与分页

`ListNiagaraSystems`、`ListNiagaraEmitters` 和 `SearchModules` 都支持稳定分页：

- `MaxResults` 必须为 1–500，默认 100。
- `Offset` 必须为非负整数，默认 0。
- 返回 `TotalCount`、`Returned`、`Offset` 和 `HasMore`；存在下一页时还会返回 `NextOffset`。
- 结果按完整资产对象路径稳定排序，因此连续分页不会依赖 Asset Registry 的枚举顺序。

`SearchModules` 只返回 Niagara 编辑器认可的可见 Module 库资产；弃用、隐藏和非库脚本不会出现在结果中，也不会为了搜索而逐个加载脚本资产。

`GetSystemDetail` 对 Emitter 和 User Parameter 分别分页：`MaxEmitters` 为 1–128，`MaxUserParameters` 为 1–256，两类 Offset 均须非负。每个 Emitter 最多返回 64 个 Renderer 和 64 个 Simulation Stage，并在截断时返回总数和标记。User Parameter 按名称/类型稳定排序，值文本最多 4096 字符；对象与 Data Interface 返回对象路径。所有工具的根输入 Schema 默认拒绝未声明字段，具有枚举、分页或动态对象的工具会进一步声明相应边界。

## 模块详情

`GetModuleDetail` 只检查所选 Spawn、Update 或 Event 阶段参数映射主链上的模块，`ModuleIndex` 与实际执行顺序一致。重名模块必须使用索引；同时传入名称与索引时，两者必须指向同一模块。存在多个 Event 阶段时必须提供 `UsageId`。

模块输入按 `MaxResults`（1–256，默认 100）和 `Offset` 分页，并返回真实 Niagara 类型及 `ValueSource`。图覆盖、链接/动态覆盖和 Rapid Iteration 值会被明确区分；`Value` 最多返回 4096 字符，并由 `bValueTruncated` 明示是否截断。未形成覆盖的输入统一标记为 `ModuleDefaultOrBinding`；为了保持 UE 5.2+ 自包含兼容，工具不依赖仅供 NiagaraEditor 内部使用的默认值拓扑 API，因此不会进一步猜测模块默认值与内部绑定。

## Emitter 编辑

`AddEmitter` 使用 NiagaraEditor 的标准复制路径生成唯一名称、重建 Emitter 节点并同步 Overview Graph。`RemoveEmitter` 和 `RenameEmitter` 要求名称唯一；增删改名均支持撤销事务并发送 System 编辑通知。

`SetEmitterProperties` 原子写入 `FVersionedNiagaraEmitterData`，每次 1–64 项：全部属性和值预校验成功后才提交，并对每项发送版本感知的 PostEditChange。支持可编辑的标量、枚举、字符串、名称和结构体；对象引用、容器、委托、临时、弃用和非编辑属性会被拒绝。`SetEmitterProperties` 与 `SetModuleParameters` 的动态对象 Schema 只接受字符串、数字或布尔值，并与运行时数量边界一致。字符串输入及动态属性名的 Schema 和运行时上限不超过 4096 字符，部分资产路径更严格；输入 JSON 还限制为 8 层、2048 个值。所有加载现有 System 的工具在成功时统一返回实际对象的规范 `SystemPath`；最终错误文本最多 4096 字符，可用名称候选也会限制数量与文本长度。

限制：UE 5.2 未导出 NiagaraEditor 私有的 merge-adapter cache 清理 API。`RemoveEmitter` 会销毁引用实例、移除并重连公开 System Graph、同步 Overview Graph 并发送编辑通知，但无法显式清该私有缓存；响应会返回 `Warnings`，在紧接着执行继承 Emitter merge 工作流前应刷新或重开 System。插件不会为此依赖引擎 Private 头或非导出符号。

## Renderer 编辑

`SetRendererType` 以单个撤销事务替换名称唯一的 Emitter 上至多 64 个 Renderer。新 Renderer 会先成功创建再移除旧项，因此创建失败不会破坏现有配置。UE 5.2+ 的公共交集类型为 Sprite、Mesh、Ribbon、Light、Decal 和 Component；仅新版本提供的 Volume 不会无条件暴露。

`SetRendererProperties` 每次原子写入 1–64 个属性：所有名称和值先在临时 Renderer 上完成预校验，再一次性提交并逐项发送 PostEditChange，以触发绑定重建和必要的编译失效。仅允许可编辑的标量、枚举、字符串、名称、结构体及类型兼容的材质引用；容器、委托、非材质对象引用、临时、弃用和非编辑属性会被拒绝。`RendererIndex` 必须非负，工具输入 Schema 也声明相同的枚举、数量和类型边界。

## User Parameter 编辑

`AddUserParameter` 以撤销事务添加 float、int、bool、Vector、Vector2、Vector4 或 Color 值参数，并发送 System PostEditChange。`SetSystemParameters` 每次原子写入 1–64 个现有值参数：所有名称和值预校验并保存旧值后才提交；JSON 布尔值会按 Niagara 的 `FNiagaraBool` 表示写入。`RemoveUserParameter` 按唯一名称执行与 UE 5.8 External 相同的公开 exposed-parameter 删除语义。

对象与 Data Interface 参数需要类型化资产或实例处理，不能安全地走文本值导入，因此 `AddUserParameter`/`SetSystemParameters` 不声称支持它们。删除参数不会猜测如何重写图引用，调用者应另行更新引用。UE 5.2–5.8 没有共同导出的 User Parameter 重命名及引用传播 API，本插件不依赖 Private 头绕过该边界，因而不提供 RenameUserParameter。

## 资产生命周期与预览

`CreateNiagaraSystem` 仅接受可写挂载点内、不带 `.ObjectName` 后缀的合法长包名，并在创建前检查磁盘包、内存包和 Asset Registry 冲突，绝不覆盖同名资产。提供 `TemplatePath` 时会真实复制模板；未提供时才使用 Niagara Factory 创建空 System。创建结果会返回规范对象路径、包名和 dirty 状态，需调用 `SaveAsset` 落盘。

`SaveAsset` 使用非致命的包名到文件名转换，拒绝只读目标；只有 `SavePackage` 成功、文件存在且包不再 dirty 时才返回成功。`OpenAsset` 会检查 Editor 与 AssetEditorSubsystem 生命周期。`CapturePreview` 只读取内存或已保存包中的缓存缩略图，不实时渲染或改写 thumbnail cache；输出固定为一张 PNG，并限制尺寸 1024×1024、原始数据 16 MiB、压缩数据 8 MiB。

## 验证边界

源码支持 UE 5.2 及以上版本。本轮仅完成静态源码核对；实际编辑器写入、保存、重开和跨版本打包仍需后续验证。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
