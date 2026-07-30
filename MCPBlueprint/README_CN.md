[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint

MCPBlueprint 是一个自包含的 Unreal Editor 插件，通过 HTTP MCP 向 AI 客户端提供蓝图发现、图表编辑、成员管理、组件管理、资产生命周期与编译反馈能力。插件支持 Unreal Engine 5.2 及以上版本，不依赖其他用户制作的插件。

## 连接方式

默认端点为 `http://127.0.0.1:8766/mcp`。若端口被占用，服务器会继续尝试后续端口，工具栏会显示实际端点。请在 AI 客户端中把该 URL 配置为 HTTP MCP 服务器。

服务器实现 `initialize`、`tools/list`、`tools/call` 和 `ping`。写操作在游戏线程执行；引擎支持时会纳入编辑器事务以便 Undo；修改只把资产标记为 Dirty，不会自动保存到磁盘。

## 工具（38 个）

### 发现与读取

| 工具 | 用途 |
|---|---|
| `ListBlueprints` | 使用稳定且有上限的分页列出蓝图资产。 |
| `GetBlueprintOverview` | 读取图表、变量、函数、组件、接口与父类。 |
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
| `AddNodePin` | 为 Sequence、容器和 Switch 等支持动态 Pin 的节点添加 Pin。 |
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
| `SetClassDefaults` | 原子设置可编辑的 Class Default Object 属性。 |

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
- 删除资产不可 Undo。未传 `bForce` 时会拒绝删除被引用资产；强制删除可能清空引用并破坏依赖资产。
- 截图文件输出限定在 `Saved/MCPBlueprint/Screenshots`；绝对路径和路径穿越会被拒绝。
- 仅源码静态兼容审查不能代替在各目标引擎版本中的真实编译与测试。

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
