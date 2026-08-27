[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint 用户教程

MCPBlueprint 通过 MCP 向 AI 客户端提供蓝图发现、图表编辑、成员、组件、资产生命周期、编译反馈和图表截图能力。本教程只说明用户操作流程；当前编辑器会话中真实可用的工具名和参数以 `tools/list` 返回结果为准。

## 连接

1. 将 MCPBlueprint 安装并启用为引擎插件。
2. 打开 **项目设置 → 插件 → MCP Blueprint**。
3. 保持 **Auto Start** 开启，或通过插件工具栏启动服务器。
4. 在 MCP 客户端中添加 `http://127.0.0.1:8766/mcp`。配置端口被占用时，MCPBlueprint 会尝试后续端口，工具栏会显示实际端点。
5. 修改资产前，先让客户端调用 `initialize`、`tools/list` 和 `ping`。

服务器只接受本机 HTTP MCP 请求。浏览器风格的 `Origin` 只允许 `localhost`、`127.0.0.1` 和 `[::1]`；普通非浏览器 MCP 客户端可以省略 `Origin`。

## 推荐流程

每次写操作都使用同一顺序：

1. **检查** — 用读取工具确定精确的资产、图、节点 GUID、Pin、成员或组件。
2. **预览** — 工具支持 `bDryRun` 时，对完全相同的目标操作先执行 dry-run，检查 blocker、受影响资产和规范化操作。
3. **应用** — 只发送一次已批准写入，不伪造节点 SpawnerId 或稳定 GUID。
4. **读回** — 再次查询目标图或资产，核对值、节点、Pin、连线或名称。
5. **编译** — 获取权威编译状态和诊断。
6. **显式保存** — 写工具会把支持的资产标脏，但不会静默保存。

函数重命名、函数签名修改、可能破坏数据的 Struct/Enum 编辑和磁盘重载等高风险操作使用显式批准参数。工具返回 blocker 代表拒绝，不代表可以绕过门禁。

## 查找资产与图表

- 用 `ListBlueprints` 分页查找蓝图资产。
- 用 `GetBlueprintOverview` 查看图表、变量、函数、组件、接口和父类。
- 用 `ListBlueprintMembers` 查看函数、自定义事件、事件分发器和局部变量。
- 修改图表前，用 `GetGraphDetail` 获取当前节点 GUID、Pin、默认值、连线和坐标。
- 写入后用 `GetCompileErrors` 获取编译状态和诊断。

始终使用插件返回的精确资产路径。大型图应按有界详情读取，不要根据截图猜测结构。

## 安全添加图表逻辑

1. 用 `GetGraphDetail` 读取目标图。
2. 用 Unreal 英文动作名调用 `SearchGraphNodes`，例如 `Branch`、`For Loop`、`For Each Loop`、`Switch on Int`、`Add`、`Subtract`、`Multiply` 或 `Divide`。
3. 把返回的 Action Registry `SpawnerId` 传给 `ApplyGraphPatch`，不得自行编造。
4. 用 `bDryRun=true` 预览完整 Patch。
5. 审阅全部 blocker 和规范化操作后，才对同一个 Patch 使用 `bDryRun=false`。
6. 读回图表、编译并显式保存。

标准流程控制节点来自 Unreal 的真实 Action Registry。应通过 `GetGraphDetail` 核对返回节点的类型和 Pin，不要把截图当作结构证据。

`ApplyGraphPatch` 可以在同一事务中有界地创建、删除、连接、断开、设置默认值、移动和布局节点。`FormatGraph` 用于整理现有图或选择集，`AddCommentBox` 用于为选中节点创建原生 Comment Box。用 `LayoutScope` 和 `LayoutStyle` 表达布局范围与风格，不要手动搬动无关节点。

### 一次调用或多次调用

可以在一次 Patch 中完成节点创建与连接。

也可以先创建节点，再用下一次 Patch 建立连接；自动布局会根据真实连接重新组织图表。

目标图已经存在兼容事件时，应复用已有事件，不要创建重复事件。

## 成员、Struct 与 Enum

- 变量：`AddVariable`、`ModifyVariable`、`RemoveVariable`。
- 函数：`CreateFunction`、`RenameFunction`、`ModifyFunctionSignature`、`RemoveFunction`。
- 事件与分发器：`CreateCustomEvent`、`AddEventNode`、`AddBoundEvent`、`AddEventDispatcher`、`RemoveEventDispatcher`。
- 局部变量：先添加或删除声明，再使用 `SearchGraphNodes` 返回的函数作用域 Local Get/Set 动作。
- User Defined Struct 与 Enum：通过稳定字段或条目标识创建、读取和修改。

先预览引用影响，再按工具响应要求提供批准参数；当操作可能丢弃序列化数据、改变 Enum 值语义或更新外部调用者时尤其如此。

## 组件与资产生命周期

组件工具用于管理 Actor 蓝图的 SCS 组件及其模板属性。添加、重命名、重设父级、复制、设置根组件或删除组件前，先读取当前组件树。

资产生命周期工具可以创建、复制、重命名、重设父类、删除、打开、关闭、重载、编译和保存支持的蓝图资产。从磁盘重载前使用 dry-run 或显式批准丢弃内存改动；编译成功不代表内存修改已经保存。

## 截取图表

只有在图表已经读回并编译后，才使用 `CaptureGraphScreenshot`。优先选择文字、Pin 和连线可读的全图或节点聚焦画面，不要为追求全景把缩放降到不可读。需要对比画幅时，复用响应中的视图位置、缩放、宽高和 Viewport Token。

本教程中的截图只用于说明最终用户操作。开发验收、历史对比、发布准备和被否决的截图不保存在插件仓库。

## 设置

打开 **项目设置 → 插件 → MCP Blueprint**：

- **Port** — 默认 `8766`；被占用时最多尝试后续九个端口。
- **Auto Start** — 随编辑器启动服务器。
- **Auto Compile After Modify** — 支持的写操作完成后自动编译。
- **Request Timeout Seconds** — 普通工具超时。
- **Compile Timeout Seconds** — 编译工具独立超时。

## 故障排查

- **无法连接：** 核对工具栏显示的端点，配置端口可能已被占用。
- **找不到工具：** 刷新 `tools/list`，不要依赖旧缓存的工具列表。
- **写入被拒绝：** 阅读返回的 blocker 或批准参数要求，重试前重新读取目标状态。
- **编译失败：** 调用编译诊断工具，根据真实蓝图错误修复后再保存。
- **图表结果异常：** 对比修改前后的 `GetGraphDetail`；适用时执行 Undo，然后重新读回。

## 支持

如有问题或反馈，请使用[统一联系页面](https://mengzhishanghun.github.io/mengzhishanghun/contact/)。
