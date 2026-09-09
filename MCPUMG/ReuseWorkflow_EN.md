# Widget Reuse Workflow

This workflow covers extracting a dependency-free widget subtree, moving portable styles between Widget Blueprints, comparing temporary states at multiple sizes, and extending UMG animations.

## Extract a component

`ExtractWidgetComponent` copies one named WidgetTree subtree into a new unsaved `/Game` Widget Blueprint. `BlueprintPath`, `WidgetName`, and new `AssetPath` are required. Start with `dryRun: true` to validate the source, subtree dependencies, and target path without creating an asset.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "WidgetName": "ItemCard",
  "AssetPath": "/Game/UI/Components/WBP_ItemCard",
  "dryRun": true
}
```

Native widgets and internal slot properties are copied. The source is not changed. The subtree may use ordinary `bIsVariable` widgets, but extraction rejects graph, BindWidget, delegate, animation, desired-focus, cross-subtree widget/navigation, Named Slot content, and UI-component dependencies. After a non-dry run, save the newly created Widget Blueprint explicitly with `SaveWidgetBlueprint`.

## Export and apply a theme

`ExportWidgetTheme` requires `BlueprintPath` and an explicit unique `WidgetNames` array of up to 64 Image, Border, TextBlock, or Button names. It exports the portable version-1 style subset that `ApplyWidgetTheme` can reapply. Layout is not exported, so applying an exported theme does not change layout.

```json
{"BlueprintPath":"/Game/UI/WBP_Source","WidgetNames":["TitleText","ConfirmButton"]}
```

`ApplyWidgetTheme` requires `BlueprintPath` and `Theme`; optional `dryRun` validates without mutation. A theme has `version: 1` and 0–64 `rules`. Each rule must exactly match a target `name` and UObject `class`; at most 256 targets may be matched. Omitted style or layout fields leave their current values unchanged.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Target",
  "dryRun": true,
  "Theme": {"version":1,"rules":[{"name":"TitleText","class":"TextBlock","style":{"Text":{"fontSize":28}}}]}
}
```

`style.Brush`, `style.Text`, and `style.Button` use the structured style fields documented in [Common styles](./CommonStyles.md); `Text.font` accepts UFont assets under `/Game` or `/Engine`, including the default Roboto font. `layout` uses the existing `SetWidgetLayout` vocabulary. Invalid resources, unknown fields, duplicate name/class pairs, missing targets, class mismatches, incompatible style channels, and invalid layout reject the entire request. A successful non-dry run is in memory only; call `SaveWidgetBlueprint` to persist it.

## Compare temporary states and sizes

`CaptureWidgetPreviewMatrix` renders the Cartesian product of required named `States` and `Sizes`. A state accepts `InstanceProperties`, `WidgetOverrides`, and `ListSamples` from `CaptureWidgetPreview`; a size has `name`, `width`, and `height`. Names must be unique within their arrays. The product is limited to 16 cells and 32 MP total; every cell also keeps the 4096-pixel edge and 8 MP limit of a normal preview.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "States": [
    {"name":"empty","ListSamples":[{"name":"ItemList","items":[]}]},
    {"name":"disabled","WidgetOverrides":[{"name":"BuyButton","properties":{"bIsEnabled":"False"}}]}
  ],
  "Sizes": [{"name":"phone","width":390,"height":844},{"name":"desktop","width":1280,"height":720}]
}
```

Each cell is a `TemporaryNewInstance`, not a PIE/Game screenshot. Asset-path list samples are copied per cell. An inline sample must use a non-abstract, non-deprecated `UDataAsset` or `UCurveBase` subclass with `Within=UObject`; Actor, component, world, game-instance, and subsystem classes are rejected. `UDataAsset` itself is abstract, so use a project-specific derived class or `/Script/Engine.CurveFloat`, for example `{"class":"/Script/Engine.CurveFloat"}`. Inline samples are created transiently; the tool does not call arbitrary functions or save them. Widget Construct and list-entry code can still have external side effects.

## Extend an animation

`DuplicateAnimation` copies an existing animation, including its MovieScene tracks and widget bindings. `BlueprintPath`, `SourceAnimationName`, and a unique `NewAnimationName` are required. `SetAnimationPlaybackRange` then sets `[StartSeconds, EndSeconds)` for an existing animation; both values must be finite, start at or above zero, and have `EndSeconds > StartSeconds`.

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","SourceAnimationName":"FadeIn","NewAnimationName":"FadeInFast"}
```

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"FadeInFast","StartSeconds":0,"EndSeconds":0.2}
```

`SetAnimationKeys` supports `TangentMode` (`Auto`, `User`, or `Break`) and finite `ArriveTangent`/`LeaveTangent` for Cubic keys. Linear and Constant keys reject tangent values. These animation changes are not saved automatically; call `SaveWidgetBlueprint` when ready.

`GetAnimationDetail` also returns Cubic tangent modes and arrive/leave slopes. `Auto` lets the engine calculate slopes; use `User` or `Break` for manual slopes. Tangents must be JSON numbers representable as float.
