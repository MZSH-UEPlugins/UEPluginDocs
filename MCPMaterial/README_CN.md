[English](./README.md) | [中文](./README_CN.md)

# MCPMaterial

MCPMaterial 为 Unreal Editor 提供面向 AI 客户端的材质与材质实例 MCP 工具。插件支持 UE 5.2 及以上版本，默认端点为 `http://127.0.0.1:8763/mcp`；端口被占用时会在有限范围内递增。

## 能力

- 发现材质、材质实例与可用表达式类。
- 读取材质图、连接、属性和 Scalar、Vector、Texture、Static Switch 参数。
- 创建材质与材质实例，批量编辑属性、参数、表达式节点和连线。
- 编译材质、保存资产、打开或关闭资产编辑器，以及捕获材质预览。
- 写入工具使用事务并标记包 Dirty；调用 `SaveAsset` 前可先调用 `CompileMaterial` 验证材质。

## 安全工作流

1. 使用 `ListMaterials`、`GetMaterialDetail`、`GetMaterialParameters` 和 `SearchMaterialExpressions` 获取真实路径、Guid、引脚与参数信息。
2. 如果目标材质或材质实例已在资产编辑器中打开，先保存或处理编辑器中的更改，再调用 `CloseAsset`。
3. 调用 `ApplyMaterialPatch`、`SetMaterialProperties`、`SetInstanceParameters`、`ResetInstanceParameter`、`RemoveConnection` 或 `FormatGraph`。
4. 调用 `CompileMaterial`，检查终态编译结果。
5. 调用 `SaveAsset` 持久化更改；需要人工检查时再调用 `OpenAsset`。

写入、编译和保存工具会拒绝操作仍在资产编辑器中打开的目标。这可避免 Material Editor 的 transient preview 与 backing asset 形成双状态，并防止编辑器稍后 Apply 时覆盖 MCP 已完成的修改。

## 重要边界

- `SetInstanceParameters` 和 `ResetInstanceParameter` 只接受可唯一定位的 Global 参数；Material Layer 与 Blend 参数需要 Association/Index 结构化寻址，当前不支持按名称写入。
- `SearchMaterialExpressions` 返回稳定的 `ClassPath` 以及输入/输出索引。短类名发生碰撞时，`ApplyMaterialPatch` 会拒绝猜测并要求精确 `ClassPath`；同名 Pin 必须使用 `[index]` 寻址。
- `ApplyMaterialPatch` 不创建或删除需要专用编辑器结构管理的 Composite、PinBase、Function Input/Output、Named Reroute 等节点。
- 创建的资产先处于内存 Dirty 状态，必须显式调用 `SaveAsset` 才会写入磁盘。
- 超时不能强制终止已被 GameThread 认领的编辑器操作；收到超时错误后应先核对编辑器状态，再决定是否重试。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
