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

## 模块详情

`GetModuleDetail` 只检查所选 Spawn、Update 或 Event 阶段参数映射主链上的模块，`ModuleIndex` 与实际执行顺序一致。重名模块必须使用索引；同时传入名称与索引时，两者必须指向同一模块。存在多个 Event 阶段时必须提供 `UsageId`。

模块输入按 `MaxResults`（1–256，默认 100）和 `Offset` 分页，并返回真实 Niagara 类型及 `ValueSource`。图覆盖、链接/动态覆盖和 Rapid Iteration 值会被明确区分。未形成覆盖的输入统一标记为 `ModuleDefaultOrBinding`；为了保持 UE 5.2+ 自包含兼容，工具不依赖仅供 NiagaraEditor 内部使用的默认值拓扑 API，因此不会进一步猜测模块默认值与内部绑定。

## Emitter 编辑

`AddEmitter` 使用 NiagaraEditor 的标准复制路径生成唯一名称、重建 Emitter 节点并同步 Overview Graph。`RemoveEmitter` 和 `RenameEmitter` 要求名称唯一；增删改名均支持撤销事务并发送 System 编辑通知。

`SetEmitterProperties` 原子写入 `FVersionedNiagaraEmitterData`，每次最多 64 项：全部属性和值预校验成功后才提交，并对每项发送版本感知的 PostEditChange。支持可编辑的标量、枚举、字符串、名称和结构体；对象引用、容器、委托、临时、弃用和非编辑属性会被拒绝。

限制：UE 5.2 未导出 NiagaraEditor 私有的 merge-adapter cache 清理 API。`RemoveEmitter` 会销毁引用实例、移除并重连公开 System Graph、同步 Overview Graph 并发送编辑通知，但无法显式清该私有缓存；响应会返回 `Warnings`，在紧接着执行继承 Emitter merge 工作流前应刷新或重开 System。插件不会为此依赖引擎 Private 头或非导出符号。

## 验证边界

源码支持 UE 5.2 及以上版本。本轮仅完成静态源码核对；实际编辑器写入、保存、重开和跨版本打包仍需后续验证。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
