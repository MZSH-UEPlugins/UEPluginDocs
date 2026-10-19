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

The result is explicitly `AuthoredIndicatorsOnly`. It reports bounded widget/depth/class counts, Canvas and Overlay complexity, property bindings, volatile widgets, RetainerBox, InvalidationBox, and ListView instances. `bTruncated`, `bCycleDetected`, and `bDuplicateReferenceDetected` describe traversal limits and references. It does not measure milliseconds, CPU, GPU, memory, paint cost, or runtime invalidation. Use `MeasureWidgetPerformance` for a live widget sample or `CaptureWidgetTrace` for bounded native tracing; follow the target and cleanup rules in the [runtime workflow](./RuntimeWorkflow_EN.md). Unreal Insights and Widget Reflector support deeper analysis.

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

Native MVVM availability and the opt-in `MCPUMGMVVM` Editor module are separate. In `GetUIFrameworkCapabilities.ModelViewViewModel`, inspect `EnhancedModule.OptInModuleDeclared`, `ModuleLoaded`, `CanWriteBindingsNow`, and `AvailabilityReason`. The base plugin does not gain binding-write tools merely because native MVVM is enabled. An enhanced build must include and load `MCPUMGMVVM` and enable the native `ModelViewViewModel` plugin; editing a package descriptor without the compiled module is insufficient. The running editor's `tools/list` is authoritative.

The enhanced module registers `GetMVVMViewDetails`, `ValidateMVVMBindingAdvanced`, `SetMVVMBinding`, `RemoveMVVMBinding`, `UpdateMVVMViewModelContext`, and `RemoveMVVMViewModelContext`. Read current context and binding GUIDs and fingerprints, then validate full ordered paths. For `SetMVVMBinding`, omit `BindingId` to create; provide both `BindingId` and current `ExpectedFingerprint` to update. `SourcePath` and `DestinationPath` are nonempty arrays of ordered field names, not dotted strings.

```json
{"BlueprintPath":"/Game/UI/WBP_Inventory","SourceType":"ViewModel","SourceName":"InventoryViewModel","SourcePath":["Title"],"DestinationWidgetName":"TitleText","DestinationPath":["Text"],"BindingMode":"OneTime"}
```

Enhanced binding tools accept `OneTime`, `OneWay`, and native `TwoWay`. Notification-driven modes require real FieldNotify descriptors. `TwoWay` also needs readable and writable fields of the same type. The native UE 5.2–5.8 compiler rejects **TwoWay with a conversion function**, so the tool rejects that combination before mutation. A one-way binding may provide `SourceToDestinationConversion` as the path of an existing loaded, valid, one-argument `UFunction`; reverse conversion is not offered. Write tools compile and attempt Undo on failure, but do not save. `RemoveMVVMBinding` needs `BindingId` and `ExpectedFingerprint`; context updates/deletion need the exact `ViewModelId` and expected old values. Deleting a referenced context requires `CascadeBindingIds` to list exactly all dependent binding IDs. Save the intended Widget Blueprint explicitly afterward.

In UE 5.2, native MVVM cannot subscribe to notifications across a nonterminal UObject property path segment. The enhanced validator and writer reject those `OneWay` and `TwoWay` paths before mutation, even if the object field still points to an instance of the same class. This restriction does not itself reject `OneTime`, root FieldNotify fields, struct-only paths, or terminal UObject fields. UE 5.3 and later provide the native mechanism, but each path and project MVVM setting still needs validation. In UE 5.3, a deliberately null dynamic source still produces native error logs; verify subscription and value updates separately after the object is restored. `GetUIFrameworkCapabilities.ModelViewViewModel.EnhancedModule.BindingLimits.DynamicObjectSourceNotifications` reports `EngineSupported`, `Reason`, and `RequiresNativePathValidation`; support for the mechanism does not verify every path or setting.

For CommonUI input assets, `ConfigureCommonUIInput` accepts `BlueprintPath`, `WidgetName`, nonempty `Properties`, and optional `dryRun`. Its allowlist is `CommonButtonBase.TriggeringEnhancedInputAction`, `CommonActivatableWidget.InputMapping`/`InputMappingPriority`, and `CommonActionWidget.EnhancedInputAction`/`InputActions`. References must name existing persistent assets. `InputChainStatus: NotVerifiedByAssetConfiguration` does not prove the complete input chain works. For live instances, get a token with `ListRuntimeWidgets`, inspect with `InspectCommonUI`, and use `ControlCommonUI` actions `Activate`, `Deactivate`, `Push`, or `Remove`. `Push` needs container `WidgetName` and compiled `WidgetClassPath`; `Remove` needs the current stacked `TargetToken`. See [runtime interaction and diagnostics](./RuntimeWorkflow_EN.md).
