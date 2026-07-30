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

## 验证边界

源码支持 UE 5.2 及以上版本。本轮仅完成静态源码核对；实际编辑器写入、保存、重开和跨版本打包仍需后续验证。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
