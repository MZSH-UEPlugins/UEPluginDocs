# CaptureWidgetPreview

列表样本使用临时资产副本，不把原资产直接交给列表条目。副本引用的外部资源仍可能共享；Widget 生命周期与列表回调可以运行项目代码，这不是脚本沙箱。用于预览的业务逻辑应无副作用。

`CaptureWidgetPreview` 会创建一个已经编译的 Widget Blueprint 的全新临时实例，应用仅用于预览的数据，并返回离屏渲染的 PNG。它是设计预览，不是正在运行的游戏 Widget 截图。

该工具不会编辑或保存 Widget Blueprint、其 WidgetTree、已有 PIE/Game 实例，也不会修改作为列表样本传入的资产。它同样不会编译 Blueprint；请先显式编译，并确保 Blueprint 状态为 `UpToDate`。

## 参数

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `BlueprintPath` | 是 | Widget Blueprint 资产路径。 |
| `Width`、`Height` | 是 | 正整数像素，每个不超过 4096，乘积不超过 8 MP。 |
| `InstanceProperties` | 否 | 临时 `UUserWidget` 实例的可编辑属性，格式为 `{ "Property": "UE 文本格式" }`。 |
| `WidgetOverrides` | 否 | 最多 64 个 `{ "name": "WidgetName", "properties": { ... } }`；属性只作用于临时树中对应的 Widget。 |
| `ListSamples` | 否 | 最多 64 个 `{ "name": "ListWidgetName", "items": ["/Game/...Asset.Asset"] }`；数据只绑定到临时 ListView/TileView/TreeView。每个列表最多 100 个项目，单次调用共最多 200 个；允许空列表。 |

每个 `WidgetOverrides` 条目都必须提供非空的 `properties`。`WidgetOverrides` 中的 Widget 名称、`ListSamples` 中的列表名称必须各自唯一。同一列表中重复的资产路径会被拒绝。每个 `properties` 映射的值都使用 Unreal 文本格式，规则与 `SetWidgetProperties` 相同。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "Width": 1280,
  "Height": 720,
  "InstanceProperties": { "Title": "预览背包" },
  "WidgetOverrides": [
    { "name": "TitleText", "properties": { "RenderOpacity": "0.8" } }
  ],
  "ListSamples": [
    { "name": "ItemList", "items": ["/Game/Items/DA_Sword.DA_Sword"] },
    { "name": "EmptyStateList", "items": [] }
  ]
}
```

响应会明确给出 `PreviewState: TemporaryNewInstance`、`RuntimeState: NotRuntime`、`ProductionState: Unchanged` 与 `SaveStatus: NotSaved`，并附带 PNG。还会返回实际 RenderTarget 尺寸、像素数、两次布局渲染，以及实际应用的 Widget 覆盖和列表样本计数。

创建任何 `UUserWidget` 都会进入其 Blueprint 构造生命周期。该工具不会调用项目特定初始化方法或任意 Blueprint 函数，但用户编写的 Construct 代码本身可能有副作用，因此无法承诺所有 Blueprint 都是纯函数。
