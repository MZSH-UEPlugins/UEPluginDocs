# Widget 主题

`ExportWidgetTheme` 从指定的 Image、Border、TextBlock 或 Button 导出可移植的版本 1 主题；`ApplyWidgetTheme` 将该主题应用到另一份具有相同控件名称和类的 Widget Blueprint。导出只读取资产；应用在一个事务中修改内存，不会自动保存。

## 导出主题

`BlueprintPath` 和唯一的 `WidgetNames` 数组为必填项，最多 64 个控件。仅导出可以完整表示的样式：Brush、Text 以及 Button 的四种状态和 padding。动态 Slate 颜色、非 `/Game` 的 Brush 资源、非 UFont 的文字资源会被拒绝，避免产生不完整的主题；文字 UFont 可以位于 `/Game` 或 `/Engine`，因此默认 Roboto 也能导出并回放。当前不导出布局；导出的样式重新应用时不会改变布局。

```json
{
  "BlueprintPath": "/Game/UI/WBP_MainMenu",
  "WidgetNames": ["PlayButton", "TitleText"]
}
```

`uv` 使用 `{"minX","minY","maxX","maxY"}` 形式；没有有效 UV 覆盖时输出 `uv: null`，重新应用时会清除目标已有的 UV 覆盖。

## 应用主题

`ApplyWidgetTheme` 需要 `BlueprintPath` 和 `Theme`。主题根对象只能是 `{"version":1,"rules":[...]}`，规则数为 0 至 64；每条规则以精确的 `name` 和 UObject `class` 定位目标。可选 `dryRun: true` 使用相同的完整预检但不修改资产。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Options",
  "dryRun": true,
  "Theme": {
    "version": 1,
    "rules": [
      {
        "name": "PlayButton",
        "class": "Button",
        "style": {
          "Button": {
            "normal": {
              "resource": null,
              "drawType": "RoundedBox",
              "imageSize": {"x": 96, "y": 36},
              "tint": {"r": 0.1, "g": 0.4, "b": 0.9, "a": 1},
              "margin": {"left": 4, "top": 4, "right": 4, "bottom": 4},
              "tiling": "NoTile",
              "uv": null
            },
            "normalPadding": {"left": 8, "top": 4, "right": 8, "bottom": 4}
          }
        }
      }
    ]
  }
}
```

`style.Brush` 支持 `resource`、`drawType`、`imageSize`、`tint`、`margin`、`tiling`、`uv` 和 `outline`；`style.Text` 支持 TextBlock 的颜色、`/Game` 或 `/Engine` UFont、字号、typeface、字距、行高、换行和省略；`style.Button` 支持 `normal`、`hovered`、`pressed`、`disabled` 及 `normalPadding`、`pressedPadding`。`layout` 使用 `SetWidgetLayout` 的既有字段。未提供的字段保留当前值。

未知字段、重复的名称/类组合、目标不存在、类不匹配、不兼容的样式通道、无效布局或无效资源都会拒绝整个请求。主题最多命中 256 个目标；非 dry-run 成功后请按需要调用 `SaveWidgetBlueprint` 保存。
