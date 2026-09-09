# Advanced UMG Workflow

This guide covers bounded batch edits, authored performance indicators, and optional CommonUI/MVVM tools. It assumes the target editor already has the optional framework enabled when you choose to use it; MCPUMG never enables plugins for you.

## Batch property updates

`BatchSetWidgetProperties` changes ordinary widget properties across up to 16 Widget Blueprints and 256 widgets. It defaults to `dryRun: true`. Every target must include the current `expectedFingerprint` returned by `ExportWidgetSpec`; duplicate targets, stale fingerprints, unsafe properties, or an invalid request reject the whole batch before a change is made.

```json
{
  "dryRun": true,
  "Targets": [
    {
      "BlueprintPath": "/Game/UI/WBP_Menu",
      "expectedFingerprint": "sha1:replace-with-current-export",
      "WidgetName": "TitleText",
      "Properties": {"RenderOpacity": "0.9"}
    }
  ]
}
```

For a non-dry run, each Widget Blueprint has its own Undo transaction and remains unsaved. The tool does not compile, save, or change slots, relationships, or structure. It verifies every written field after the write. Per-blueprint status is `Applied`, `NoChange`, `Failed`, or `NotAttempted`; a failure does not claim cross-blueprint rollback. Use the editor's Undo for an applied blueprint, or save each intended result explicitly with `SaveWidgetBlueprint`.

## Authored performance indicators

`InspectWidgetPerformance` reads a design-time WidgetTree without creating an instance or changing the asset. `MaxWidgets` is an optional integer from 1 to 1000 (default 500).

```json
{"BlueprintPath":"/Game/UI/WBP_Inventory","MaxWidgets":500}
```

The result is explicitly `AuthoredIndicatorsOnly`. It reports bounded widget/depth/class counts, Canvas and Overlay complexity, property bindings, volatile widgets, RetainerBox, InvalidationBox, and ListView instances. `bTruncated`, `bCycleDetected`, and `bDuplicateReferenceDetected` describe traversal limits and references. It does not measure milliseconds, CPU, GPU, memory, paint cost, or runtime invalidation. Use Unreal Insights and Widget Reflector when you need measured runtime evidence.

## Optional CommonUI

Call `GetUIFrameworkCapabilities` first. It reports whether CommonUI is not installed, disabled, enabled but not loaded, or available, and does not change module or project state. Enable CommonUI manually in your project and restart the editor if you choose to use it.

`ConfigureCommonUIWidget` edits one existing supported widget in a Widget Blueprint. It accepts `BlueprintPath`, `WidgetName`, `Properties`, and optional `dryRun`.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Menu",
  "WidgetName": "ConfirmButton",
  "Properties": {"bIsLocked": "True"},
  "dryRun": true
}
```

Supported families are CommonButtonBase (style, lock/selection/toggle behavior, click/touch/key behavior, and legacy TriggeringInputAction), CommonActivatableWidget (back, activation, focus/modal behavior, and activation/deactivation visibility), and CommonTextBlock (style, scroll style, and mobile font-size multiplier). The tool does not call CommonUI functions, modify style defaults, configure Enhanced Input, enable plugins, or save the asset.

## Optional MVVM

`GetMVVMView` reads existing MVVM contexts and bindings. `AddMVVMViewModelContext` adds one real context through the enabled MVVM editor subsystem:

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "ViewModelClass": "/Game/UI/VM/BP_InventoryViewModel.BP_InventoryViewModel_C",
  "CreationType": "CreateInstance"
}
```

Only `Manual` and `CreateInstance` are accepted. The class must be a current concrete non-Widget class implementing `NotifyFieldValueChanged` and allow the requested creation policy. The tool does not save; inspect the result with `GetMVVMView`, then explicitly compile and save when appropriate.

`ValidateMVVMBinding` is read-only. It validates one direct, exact-type `OneTime` relationship from a ViewModel context or source widget to a writable runtime property on an existing destination widget.

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "SourceType": "ViewModel",
  "SourceName": "InventoryViewModel",
  "SourceProperty": "Title",
  "DestinationWidgetName": "TitleText",
  "DestinationProperty": "Text",
  "BindingMode": "OneTime"
}
```

It does not add, remove, or modify bindings. Functions, nested paths, static arrays, conversion functions, implicit conversions, private/editor-only properties, and notification-driven modes are outside this validator.
