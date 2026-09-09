# Widget 性能结构诊断

`InspectWidgetPerformance` 只读取一个 Widget Blueprint 的设计态 WidgetTree，并汇总可能影响 UI 复杂度的结构性信号。它不会创建 Widget、进入 PIE、编译、保存或修改资产。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "MaxWidgets": 500
}
```

`BlueprintPath` 为必填 Widget Blueprint 资产路径。`MaxWidgets` 可选，默认 `500`，必须是 `1` 到 `1000` 的 JSON 整数；不接受小数、字符串或布尔值。达到上限时返回 `bTruncated: true`，这表示统计只覆盖已遍历的前缀，不能将其视为全树总数。

## 返回内容

- `MeasurementStatus` 固定为 `AuthoredIndicatorsOnly`，说明结果是已编辑结构的信号，不是实测性能。
- `WidgetsAnalyzed`、`MaxDepth` 与 `WidgetClassCounts`：已分析节点数、最大树深度和控件类计数。
- `PanelComplexity`：Canvas Panel 与 Overlay 的实例数，以及各自单个容器的最大直接子控件数。
- `PropertyBindingCount`：Widget Blueprint 中的属性绑定条目数。
- `VolatileWidgetCount`、`RetainerBoxCount`、`InvalidationBoxCount`、`ListViewCount`：对应控件或标记的实例数。
- `bCycleDetected`：Widget 在当前递归路径中再次出现时为 `true`。工具会停止该路径，避免循环导致无界遍历。
- `bDuplicateReferenceDetected`：已经统计过的 Widget 又从另一条路径可达时为 `true`。共享引用会去重，不会被误报为循环。
- `bTruncated`：分析节点达到 `MaxWidgets` 时为 `true`。

这些指标不能给出毫秒、CPU、GPU、内存、Slate Tick/Paint 开销或实际运行时失效行为。它们用于找出值得进一步观察的结构，例如过深的层级、大量直接子控件、过多绑定或易失控件。需要实测时，请在目标场景中使用 Unreal Insights；需要检查实际 Slate 层级、无效化和绘制行为时，请使用 Widget Reflector。
