# 通用控件样式

`SetWidgetStyle` 的 `Brush`、`Text`、`Button` 接口保持不变。所有对象会在开启事务前完整解析；未知字段、JSON 类型不符、资源不存在或资源类型不匹配都会失败，且不修改目标控件。

## 扩展控件

CheckBox、Slider、ProgressBar、EditableTextBox、ScrollBar、ComboBoxString 使用 `Style.Extended`。字段名是引擎对应 `WidgetStyle` 或控件自身的公开反射字段名；仅支持 `FSlateBrush`、`FSlateColor`/`FLinearColor`、`float`、`bool`。例如：

```json
{"Extended":{"CheckedImage":{"drawType":"Box"},"ForegroundColor":{"r":0.2,"g":0.4,"b":0.6}}}
```

常用 Brush 包括 CheckBox 的 `UncheckedImage`/`CheckedImage`，Slider 的 `NormalBarImage`/`NormalThumbImage`，ProgressBar 的 `BackgroundImage`/`FillImage`，EditableTextBox 的 `BackgroundImageNormal`/`BackgroundImageFocused`，ScrollBar 的 `NormalThumbImage`，以及 ComboBoxString 可直接反射的样式字段。Slider 的 `Value`、`MinValue`、`MaxValue`、`StepSize` 和 ProgressBar 的 `Percent`/`FillColorAndOpacity` 也可在该对象中设置。复杂嵌套样式（例如 ComboBox 的内部 ButtonStyle）不在本接口范围内。

## 文本与富文本限制

TextBlock 的 `Text` 支持 `color`、`fontSize`、已有 `/Game` 或 `/Engine` **UFont** 的 `font`、`typefaceFontName`、`letterSpacing`、`lineHeight`、`wrapTextAt`、`autoWrap` 和 `ellipsis`。`ellipsis:false` 明确设置原生 Clip 策略。裸 `FontFace` 被拒绝：它不实现 Slate 所需的 `IFontProviderInterface`，会退回 fallback 字体。引擎原生 UFont（例如默认 Roboto）可用于样式和主题回放。

RichTextBlock 使用 `Style.RichText`：`textStyleSet` 必须是已有 `/Game` DataTable，且行结构**严格等于**引擎 `FRichTextStyleRow`；`decoratorClasses` 是已有、非 abstract `URichTextBlockDecorator` 子类的路径数组。工具仅引用并调用原生 `SetTextStyleSet`/`SetDecorators`，不会创建业务 Decorator。例如：

```json
{"RichText":{"textStyleSet":"/Game/UI/Styles/DT_RichText","decoratorClasses":["/Game/UI/Decorators/BP_LinkDecorator.BP_LinkDecorator_C"]}}
```

`ImportUIFont` 将一个本地 `.ttf`/`.otf` 导入为新的 `/Game` FontFace，并创建同名后缀 `_Font` 的 UFont；UFont 默认 typeface 持久引用该 FontFace，返回 `FontFacePath` 与可直接用于 `Text.font` 的 `FontPath`。`SourceFile`、`AssetPath` 为必填项，64 MiB 为源文件上限；两个目标包只要已存在、已加载为非空包或磁盘已存在即拒绝，避免覆盖或夹带保存。`bSave` 必须为布尔值，默认为 `false`；为真时精确保存这两个包。默认导入后可用 `SaveUIAsset` 分别按 FontFace/UFont 路径保存单包。工具使用 UE 5.2 至 5.8 实际提供的 `UFontFileImportFactory`（其基类 `FactoryCreateFile` 最终调用 FontFace 的原生二进制工厂）。
