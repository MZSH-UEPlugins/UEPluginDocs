# Animation workflow
## Generic property tracks

`AddAnimationTrack`, `SetAnimationKeys`, and `RemoveAnimationTrack` resolve `PropertyPath` against the target object's reflected properties. `TargetKind` defaults to `"Widget"`. Use `"Slot"` for a property of the widget's current PanelSlot, such as `HorizontalBoxSlot.Padding`. Optional `SlotWidgetName` must match the current slot object's name. A root widget has no slot. After reparenting a widget into another slot, repair the old binding before editing; the tools reject it.

| Property type | `Value` format |
|---|---|
| `float` | A finite number, such as ProgressBar's `Percent` |
| `double` | A finite number, preserving double precision |
| `int32` | An integer from -2147483648 to 2147483647; step interpolation, with `Interp` omitted or set to `Constant`, and no tangent fields |
| Standalone 2D/3D/4D vector | A complete `{X,Y}`, `{X,Y,Z}`, or `{X,Y,Z,W}` object; missing fields, extra fields, and scalar values are rejected |
| `bool` | `true` or `false` |
| `uint8` | An integer from 0 to 255 |
| Enum stored as `uint8` | A name from that property's actual enum, such as `Hidden` for `Visibility` |
| `FLinearColor`, `FColor`, `FSlateColor` | `{R,G,B,A}` with linear floating point components; `A` is optional and defaults to 1 |
| `FMargin` | `{Left,Top,Right,Bottom}`, with all four fields required |
| `FWidgetTransform` component | `.Translation`, `.Scale`, and `.Shear` take `{X,Y}` or one number for both axes; `.Angle` takes a number in degrees |

Paths may include fixed struct fields and are stored with their actual property names. Arrays, sets, maps, fixed arrays, UObject traversal, deprecated fields, and integer properties other than `int32` and `uint8` are unsupported. Playback also requires a matching engine accessor, a native setter, a compatible `Set<Property>` function, or an `Interp` field that is safe to write directly. Nested fields are rejected if playback could call an unrelated root setter with the same leaf name or bypass a parent property's refresh logic. A writable field alone does not qualify it for animation.

Vector types are identified by their reflected structures: `FVector2f`, `FVector3f`, and `FVector4f` use float channels; `FVector2D`, `FVector`, `FVector4`, and the explicit `FVector3d` and `FVector4d` structures use double channels. Float components must fit in float; double components must be finite. Existing vector tracks, sections, and channel layouts must match the property dimension. Existing integer channels with linear interpolation enabled are rejected without repair.

For a bool whose name starts with lowercase `b`, a native setter or full-path accessor is required if either `Set` plus the full property name or `Set` plus the name without `b` exists. The tools reject this fallback to keep playback consistent across supported engine versions. If neither function exists, the property may use the direct `Interp` field route. Uppercase `B` is kept: `Busy`, for example, uses `SetBusy`, whose signature must be compatible.

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"Intro","WidgetName":"Progress","PropertyPath":"Percent","Keys":[{"Time":0,"Value":0},{"Time":1,"Value":1,"Interp":"Linear"}]}
```

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"Intro","WidgetName":"Progress","TargetKind":"Slot","PropertyPath":"Padding","Keys":[{"Time":0,"Value":{"Left":0,"Top":0,"Right":0,"Bottom":0}},{"Time":1,"Value":{"Left":20,"Top":0,"Right":20,"Bottom":0}}]}
```

Create a track with `AddAnimationTrack`, then call `SetAnimationKeys` with the same target and path. `bReplaceAll` defaults to `true` and replaces keys in the addressed channels; `false` merges keys. The four `FWidgetTransform` components share one parent track. Replacing one component preserves keys in the others. Calling `RemoveAnimationTrack` with any component path removes the whole parent track. Existing `RenderTransform` component paths and `CanvasPanelSlot.LayoutData.Offsets` remain supported.

A full property path must identify one track within its binding. Duplicate tracks, mismatched types or enums, multiple sections, and incompatible channels are rejected before mutation. Add, Set, and Remove support Undo/Redo and do not save assets. Save the Blueprint explicitly after editing.


`DuplicateAnimation` deep-copies the source animation's MovieScene, tracks, and widget bindings. The source stays unchanged, and the new name must be unique. `SetAnimationPlaybackRange` sets `[StartSeconds, EndSeconds)` in seconds after validating all parameters, in one transaction. Neither tool saves the asset.

Cubic keys in `SetAnimationKeys` accept `TangentMode` (`Auto`, `User`, or `Break`) and finite `ArriveTangent` and `LeaveTangent` values. Linear or Constant keys with tangents are rejected without changing the asset. `GetAnimationDetail` returns cubic tangent modes and slopes. The engine calculates `Auto` tangents; use `User` or `Break` for manual slopes. Tangents must be JSON numbers within the float range.

`GetAnimationDetail` returns the new channels in `DoubleChannels` and `IntegerChannels`; vector tracks also include `Dimensions`. These channels share the pagination budget with other tracks.

`GetAnimationDetail` pages through binding GUIDs, track and section indices, and Float, Double, Integer, Bool, and Byte channel keys. Use these addresses with `EditAnimationKeys` to delete, clear, or shift keys in one channel. Binding requests need `BindingGuid`, `WidgetName`, `TrackIndex`, `SectionIndex`, `ChannelType`, and `ChannelIndex`; slot bindings also need `SlotWidgetName`. `Operation` accepts `Delete`, `Clear`, or `Shift`. Select integer tick frames with `Frames`, or an inclusive range with both `StartSeconds` and `EndSeconds`. `Clear` without a selector empties the channel. `Shift` also requires `ShiftSeconds`, and destinations must avoid existing keys and stay inside the playback range.

`bDryRun` defaults to `true` and returns the current section's `Fingerprint` and affected key count. To apply, send `bDryRun:false` and that value as `ExpectedFingerprint`. A stale fingerprint is rejected. Save the Blueprint explicitly after editing.

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","SourceAnimationName":"FadeIn","NewAnimationName":"FadeInFast"}
```

`RenameAnimation` takes `BlueprintPath`, the existing `AnimationName`, and a unique `NewAnimationName`. The new name must not conflict with a widget, another animation, or a Blueprint member. Renaming is rejected if a native parent `BindWidgetAnim` or `BindWidgetAnimOptional` property uses the old name, a known external asset references the Widget Blueprint, or a graph string pin contains the old animation name. Review those dependencies first. Save explicitly afterward.

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"FadeIn","NewAnimationName":"FadeInFast"}
```

## Editing event keys

`GetAnimationDetail.RootTracks` lists animation root tracks. Event track sections include `EventChannels`, `ChannelIndex`, each key's `Frame` and `Time`, and the event name, endpoint GUID, and compilation status when available. `TrackIndex` is the original index in the root track list. Do not renumber a list filtered to event tracks. Binding tracks and root tracks share `Offset`, `MaxItems`, and `NextOffset`. A parent marked `ContextOnly:true` supplies the address of a child returned on that page.

To edit a root event channel, explicitly send `TrackScope:"Root"` and `ChannelType:"Event"` with the returned `TrackIndex`, `SectionIndex`, and `ChannelIndex`. Omit `BindingGuid`, `WidgetName`, and `SlotWidgetName` entirely; even empty strings are rejected. Omitting `TrackScope` keeps the default `Binding` behavior for Float, Double, Integer, Bool, and Byte channels and their binding addresses.

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"Intro","TrackScope":"Root","TrackIndex":0,"SectionIndex":0,"ChannelType":"Event","ChannelIndex":0,"Operation":"Shift","StartSeconds":0.25,"EndSeconds":0.25,"ShiftSeconds":0.25,"bDryRun":true}
```

This selects the event at 0.25 seconds and previews moving it to 0.5 seconds. Apply the same request with `bDryRun:false` and the returned `Fingerprint` as `ExpectedFingerprint`. Repeat the dry run if tracks or sections are replaced, addresses are reordered, the playback range changes, or section contents change.

`Delete` removes selected events. `Clear` without a selector empties that event channel. `Frames` uses integer tick frames, and time ranges include both endpoints. Shift destinations must avoid unselected keys and stay inside the playback range. Edits preserve the remaining events' endpoints and payloads and support Undo/Redo. The tool does not save assets; save the Blueprint explicitly after checking the result.