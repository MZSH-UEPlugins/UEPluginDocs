# Animation workflow

`DuplicateAnimation` deep-copies the source animation's MovieScene, tracks, and widget bindings. The source stays unchanged, and the new name must be unique. `SetAnimationPlaybackRange` sets `[StartSeconds, EndSeconds)` in seconds after validating all parameters, in one transaction. Neither tool saves the asset.

Cubic keys in `SetAnimationKeys` accept `TangentMode` (`Auto`, `User`, or `Break`) and finite `ArriveTangent` and `LeaveTangent` values. Linear or Constant keys with tangents are rejected without changing the asset. `GetAnimationDetail` returns cubic tangent modes and slopes. The engine calculates `Auto` tangents; use `User` or `Break` for manual slopes. Tangents must be JSON numbers within the float range.

`GetAnimationDetail` pages through binding GUIDs, track and section indices, and Float, Bool, and Byte channel keys. Use these addresses with `EditAnimationKeys` to delete, clear, or shift keys in one channel. Binding requests need `BindingGuid`, `WidgetName`, `TrackIndex`, `SectionIndex`, `ChannelType`, and `ChannelIndex`; slot bindings also need `SlotWidgetName`. `Operation` accepts `Delete`, `Clear`, or `Shift`. Select integer tick frames with `Frames`, or an inclusive range with both `StartSeconds` and `EndSeconds`. `Clear` without a selector empties the channel. `Shift` also requires `ShiftSeconds`, and destinations must avoid existing keys and stay inside the playback range.

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

To edit a root event channel, explicitly send `TrackScope:"Root"` and `ChannelType:"Event"` with the returned `TrackIndex`, `SectionIndex`, and `ChannelIndex`. Omit `BindingGuid`, `WidgetName`, and `SlotWidgetName` entirely; even empty strings are rejected. Omitting `TrackScope` keeps the default `Binding` behavior for Float, Bool, and Byte channels and their binding addresses.

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"Intro","TrackScope":"Root","TrackIndex":0,"SectionIndex":0,"ChannelType":"Event","ChannelIndex":0,"Operation":"Shift","StartSeconds":0.25,"EndSeconds":0.25,"ShiftSeconds":0.25,"bDryRun":true}
```

This selects the event at 0.25 seconds and previews moving it to 0.5 seconds. Apply the same request with `bDryRun:false` and the returned `Fingerprint` as `ExpectedFingerprint`. Repeat the dry run if tracks or sections are replaced, addresses are reordered, the playback range changes, or section contents change.

`Delete` removes selected events. `Clear` without a selector empties that event channel. `Frames` uses integer tick frames, and time ranges include both endpoints. Shift destinations must avoid unselected keys and stay inside the playback range. Edits preserve the remaining events' endpoints and payloads and support Undo/Redo. The tool does not save assets; save the Blueprint explicitly after checking the result.