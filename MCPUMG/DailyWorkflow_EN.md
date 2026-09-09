# Daily UMG Workflow

List sample roots are temporary asset copies. Referenced external resources may remain shared. Widget lifecycle and list-entry callbacks can run project code; previews are not a script sandbox and require side-effect-free project logic.

This guide covers the everyday MCPUMG workflow for styling, nested widgets, navigation, temporary previews, and fonts. It assumes MCPUMG is connected to an open Unreal Editor and that the Widget Blueprint paths and widget names already exist.

## 1. Style a widget

Use `SetWidgetStyle` for Image, Border, TextBlock, Button, and supported extended controls. The tool validates the complete request before editing the target widget. Unknown fields, wrong JSON types, missing resources, and resource-type mismatches are rejected.

For a `TextBlock`, `Text` supports `color`, `fontSize`, an existing `/Game` `UFont` in `font`, `typefaceFontName`, `letterSpacing`, `lineHeight`, `wrapTextAt`, `autoWrap`, and `ellipsis`. Set `ellipsis: false` to select the native Clip policy. A bare `FontFace` is not accepted for `Text.font`; it is not a Slate font provider.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "WidgetName": "TitleText",
  "Style": {
    "Text": {"fontSize": 28, "font": "/Game/UI/Fonts/FF_Display_Font.FF_Display_Font", "autoWrap": true}
  }
}
```

For CheckBox, Slider, ProgressBar, EditableTextBox, ScrollBar, and ComboBoxString, use `Style.Extended`. It accepts public `WidgetStyle` or widget reflection fields of type `FSlateBrush`, `FSlateColor`/`FLinearColor`, `float`, or `bool`.

```json
{"Extended":{"CheckedImage":{"drawType":"Box"},"ForegroundColor":{"r":0.2,"g":0.4,"b":0.6}}}
```

Common examples include `UncheckedImage` and `CheckedImage` on CheckBox, `NormalBarImage` and `NormalThumbImage` on Slider, `BackgroundImage` and `FillImage` on ProgressBar, and `BackgroundImageNormal` and `BackgroundImageFocused` on EditableTextBox. Slider values (`Value`, `MinValue`, `MaxValue`, `StepSize`) and ProgressBar values (`Percent`, `FillColorAndOpacity`) are also supported. Nested styles such as ComboBox's internal ButtonStyle are outside this interface.

For `RichTextBlock`, use `Style.RichText`. `textStyleSet` must point to an existing `/Game` DataTable with exactly the engine `FRichTextStyleRow` row structure; pass `null` to clear it. `decoratorClasses` is an array of existing, non-abstract `URichTextBlockDecorator` subclass paths; pass `[]` to clear all decorators. MCPUMG references these native resources; it does not create project decorators.

```json
{"RichText":{"textStyleSet":"/Game/UI/Styles/DT_RichText","decoratorClasses":["/Game/UI/Decorators/BP_LinkDecorator.BP_LinkDecorator_C"]}}
```

To clear both settings:

```json
{"RichText":{"textStyleSet":null,"decoratorClasses":[]}}
```

## 2. Import and use a font

`ImportUIFont` imports one absolute local `.ttf` or `.otf` file as a new `/Game` FontFace and also creates a persistent companion UFont at `<AssetPath>_Font`. The UFont composite default typeface references the FontFace. `SourceFile` and `AssetPath` are required. The source must exist, be no larger than 64 MiB, and `AssetPath` must be a new valid `/Game` package path including its asset name. Both target packages must be new and empty: existing assets, on-disk packages, and non-empty loaded packages are rejected. `bSave` is optional, must be boolean, and defaults to `false`.

```json
{
  "SourceFile": "D:/Fonts/Display.otf",
  "AssetPath": "/Game/UI/Fonts/FF_Display",
  "bSave": true
}
```

The response returns `FontFacePath` and `FontPath`. Use `FontPath` in `Text.font`; `FontFacePath` is the source held by the companion UFont and cannot be assigned directly. The generated UFont has one default typeface entry; create or edit another UFont when the design needs a different multi-face typeface or broader glyph coverage.

When an import with the default `bSave: false` is ready to persist, use `SaveUIAsset` once for the returned `FontFacePath` and then once for the returned `FontPath`:

```json
{"AssetPath":"/Game/UI/Fonts/FF_Display.FF_Display"}
```

```json
{"AssetPath":"/Game/UI/Fonts/FF_Display_Font.FF_Display_Font"}
```

`SaveUIAsset` saves exactly one top-level `/Game` Texture2D, FontFace, or UFont package. `ExpectedRevision` is required only for Texture2D and must match its current content fingerprint; FontFace and UFont have no revision parameter in this version. PIE, transient or non-project packages, read-only files, shared packages, unsupported asset types, and revision conflicts are rejected. Alternatively, `bSave: true` on `ImportUIFont` saves the new FontFace package first and the companion UFont package second; this two-package operation is not a disk-atomic transaction.

## 3. Configure nested widgets and Named Slots

Use `SetWidgetInstanceProperties` to edit fields on one nested UserWidget instance in its owning Widget Blueprint. It changes the selected instance template only: it does not change the referenced Widget Blueprint's class defaults, call arbitrary functions, or write Data Assets. `BlueprintPath`, `WidgetName`, and a non-empty `Properties` object are required; values use Unreal text format.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Page",
  "WidgetName": "CardInstance",
  "Properties": {"Title": "Alchemy", "AccentColor": "(R=0.15,G=0.8,B=0.55,A=1.0)"}
}
```

Only editor-writable fields that pass `CanEditChange` are accepted; `ExposeOnSpawn` is not required. `ResetToClassDefault: true` starts each supplied field from the nested widget class default before importing the supplied text. Empty properties, structural fields, and a request that changes no value are rejected. Save the owning Blueprint with `SaveWidgetBlueprint` when ready.

Use `SetNamedSlotContent` to create a widget in one empty Named Slot. Required fields are `BlueprintPath`, `HostWidgetName`, `SlotName`, `WidgetClass`, and `WidgetName`; optional `bIsVariable` defaults to `true`, and optional `dryRun` defaults to `false`.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Page",
  "HostWidgetName": "DetailsArea",
  "SlotName": "Header",
  "WidgetClass": "TextBlock",
  "WidgetName": "Txt_DetailsHeader",
  "bIsVariable": true,
  "dryRun": true
}
```

The host must implement `INamedSlotInterface`; native hosts and compiled nested UserWidget hosts are supported. Start with `dryRun: true` to check the host, slot, class, name, compiled state, and circular references. The tool refuses occupied slots and cannot clear, replace, move, or destroy existing content. Inspect the result, then use `SaveWidgetBlueprint` explicitly.

## 4. Set keyboard or gamepad navigation

`GetWidgetNavigation` reads all six native directions (`Up`, `Down`, `Left`, `Right`, `Next`, `Previous`) for one widget:

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","WidgetName":"PlayButton"}
```

A widget without a navigation object reports `Escape` for every direction.

`SetWidgetNavigation` changes only the directions present in its non-empty `Rules` object. Each rule is `Escape`, `Stop`, or `Wrap`; `Explicit` also requires a different widget in the same WidgetTree as `TargetWidget`; `Custom` requires `CustomFunction` with the generated Widget Blueprint signature `UWidget* Function(EUINavigation)`.

```json
{
  "BlueprintPath":"/Game/UI/WBP_Menu",
  "WidgetName":"PlayButton",
  "Rules":{"Down":{"Rule":"Explicit","TargetWidget":"OptionsButton"},"Next":{"Rule":"Wrap"}},
  "dryRun":true
}
```

Set `dryRun` to `false` (or omit it) to write the validated directions in one transaction. The tool does not send input, change runtime focus, compile, or save. Use `SaveWidgetBlueprint` to persist it. Invalid directions, self/cross-tree targets, and unavailable custom functions are rejected before any modification.

## 5. Render a temporary preview

`CaptureWidgetPreview` creates a fresh temporary instance of an already compiled, `UpToDate` Widget Blueprint and returns an off-screen PNG. It never edits or saves the Blueprint, its WidgetTree, existing PIE/Game instances, or assets used as samples. It does not compile Blueprints.

`BlueprintPath`, `Width`, and `Height` are required. Width and height are positive integers up to 4096, and together may not exceed 8 megapixels. Optional `InstanceProperties` maps editable temporary UUserWidget properties to Unreal text-format strings. `WidgetOverrides` has at most 64 `{ "name", "properties" }` entries, each with non-empty properties. `ListSamples` has at most 64 `{ "name", "items" }` entries; each list allows up to 100 existing asset paths and the request allows 200 items total. Empty lists are valid; names within each array must be unique and duplicate paths within a list are rejected.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "Width": 1280,
  "Height": 720,
  "InstanceProperties": {"Title": "Inventory preview"},
  "WidgetOverrides": [{"name": "TitleText", "properties": {"RenderOpacity": "0.8"}}],
  "ListSamples": [{"name": "ItemList", "items": ["/Game/Items/DA_Sword.DA_Sword"]}, {"name": "EmptyStateList", "items": []}]
}
```

Creating a temporary UUserWidget runs its Blueprint construction lifecycle. MCPUMG does not call project-specific configuration or arbitrary Blueprint functions, but user-authored Construct logic may still have side effects. Use `CaptureRuntimeWidgetScreenshot` when you need an image of an existing runtime widget instead.
