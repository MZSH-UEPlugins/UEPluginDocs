# CaptureWidgetPreviewMatrix

`CaptureWidgetPreviewMatrix` 以 `States × Sizes` 生成一组设计预览 PNG。每个格子都会创建全新的临时 Widget 实例，因此不同状态不会相互污染；结果按状态名、尺寸名排序，`Cells[].PngIndex` 对应返回图片数组的索引。

`States` 与 `Sizes` 必填且非空，笛卡尔积最多 16 格，累计像素最多 32 MP；每张图片仍受 `CaptureWidgetPreview` 的 4096 单边和 8 MP 限制。每个状态可包含同名工具已有的 `InstanceProperties`、`WidgetOverrides` 和 `ListSamples`。

`ListSamples.items` 支持安全顶层资产路径字符串，或 `{ "class": "...", "properties": { ... } }` 动态样本。动态类必须是非 abstract、未废弃、`Within=UObject` 的 `UDataAsset` 或 `UCurveBase` 派生类；Actor、ActorComponent、World、GameInstance、Subsystem 等类型一律拒绝。`UDataAsset` 本身是 abstract，必须提供项目中的具体子类；也可以使用 `/Script/Engine.CurveFloat`。属性先在 CDO 验证，再通过 `NewObject` 在 transient package 创建对象。工具不会调用任意函数、保存对象或写入业务资产。已有样本会逐格复制到 transient package，避免条目逻辑直接修改源资产或跨状态共享同一实例。

这不是递归隔离：资产的外部引用、以及 Widget Construct/列表条目代码对外部系统产生的副作用仍不能由本工具阻止。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "States": [
    { "name": "empty", "ListSamples": [{ "name": "ItemList", "items": [] }] },
    { "name": "populated", "ListSamples": [{ "name": "ItemList", "items": ["/Game/Items/DA_Sword.DA_Sword"] }] },
    { "name": "curve", "ListSamples": [{ "name": "ItemList", "items": [{ "class": "/Script/Engine.CurveFloat" }] }] },
    { "name": "disabled", "WidgetOverrides": [{ "name": "BuyButton", "properties": { "bIsEnabled": "False" } }] }
  ],
  "Sizes": [
    { "name": "phone", "width": 390, "height": 844 },
    { "name": "desktop", "width": 1280, "height": 720 }
  ]
}
```

这仍是 `TemporaryNewInstance` 设计态预览，不代表 PIE/Game 运行状态。当前阶段不接受动画时间参数，避免在跨 UE 版本时把未实际评估的动画帧误报为截图结果。
