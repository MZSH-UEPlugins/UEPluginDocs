[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint

MCPBlueprint 是一个自包含的 Unreal Editor 插件，通过 HTTP MCP 向 AI 客户端提供蓝图发现、图表编辑、成员管理、组件管理、资产生命周期与编译反馈能力。插件支持 Unreal Engine 5.2 及以上版本，不依赖其他用户制作的插件。

## 连接方式

默认端点为 `http://127.0.0.1:8766/mcp`。若端口被占用，服务器会继续尝试后续端口，工具栏会显示实际端点。请在 AI 客户端中把该 URL 配置为 HTTP MCP 服务器。

服务器实现 `initialize`、`tools/list`、`tools/call` 和 `ping`。写操作在游戏线程执行；引擎支持时会纳入编辑器事务以便 Undo；修改只把资产标记为 Dirty，不会自动保存到磁盘。

请求必须使用 JSON-RPC `2.0`，且 `params` 与工具 `arguments` 必须是对象。浏览器风格的 `Origin` 仅允许 `localhost`、`127.0.0.1` 或 `[::1]`；非浏览器客户端可以不发送 `Origin`。

## 工具（41 个）

### 发现与读取

| 工具 | 用途 |
|---|---|
| `ListBlueprints` | 使用稳定且有上限的分页列出蓝图资产。 |
| `GetBlueprintOverview` | 读取图表、变量、函数、组件、接口与父类。 |
| `ListBlueprintMembers` | 使用统一稳定分页列出函数、Custom Event、Dispatcher 与局部变量。 |
| `GetGraphDetail` | 读取图表节点、Pin、默认值和连接。 |
| `SearchGraphNodes` | 搜索蓝图动作并返回稳定的 `SpawnerId`。 |
| `GetCompileErrors` | 编译并返回蓝图权威状态与诊断。 |

### 变量、函数与事件

| 工具 | 用途 |
|---|---|
| `AddVariable` / `ModifyVariable` / `RemoveVariable` | 管理蓝图成员变量。 |
| `CreateFunction` | 创建普通函数或重写函数，并严格校验输入/输出签名。 |
| `CreateCustomEvent` / `AddEventNode` / `AddBoundEvent` | 添加自定义事件、引擎事件、组件事件或关卡 Actor 事件。 |
| `AddLocalVariable` | 添加函数局部变量。 |
| `AddNodePin` / `RemoveNodePin` | 为 Sequence、容器和 Switch 等节点添加或删除受支持的动态 Pin。 |
| `AddInterface` | 实现蓝图接口。 |
| `AddEventDispatcher` | 创建可选签名的多播事件分发器。 |

### 图表编辑

| 工具 | 用途 |
|---|---|
| `ApplyGraphPatch` | 在一个声明式事务中执行有上限的节点创建/删除、连接/断开、默认值和布局修改。 |
| `SetPinDefaults` | 为现有输入 Pin 设置经过校验的字面量默认值。 |
| `FormatGraph` | 自动排列整张图或指定节点集合。 |

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
| `SaveAsset` / `OpenAsset` / `CloseAsset` | 保存独立蓝图资产或控制其编辑器。 |
| `DeleteAsset` / `RenameAsset` / `DuplicateAsset` | 管理独立蓝图资产。 |
| `CaptureGraphScreenshot` | 返回 PNG，也可保存到 `Saved/MCPBlueprint/Screenshots` 下。 |

## 安全与行为边界

- 新建、移动或复制蓝图时，目标必须是 `/Game` 下合法的独立包路径；若内存或磁盘已有同名包会拒绝操作。
- 关卡蓝图可以通过对象路径读取和编辑图表，但 `SaveAsset`、`DeleteAsset`、`RenameAsset` 与 `DuplicateAsset` 不处理关卡蓝图，因为这些操作会影响所属关卡包；请通过 World Editor 工作流处理关卡本身。
- `CloseAsset` 会返回之前是否打开，并核验蓝图及其子对象的所有编辑器是否真正关闭；用户取消关闭时返回错误。
- 成员变量和局部变量的默认值会在修改前按 Unreal K2 Pin 规则校验；不合法的 UE 文本值会被拒绝。
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

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
