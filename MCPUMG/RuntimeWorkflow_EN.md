# Runtime interaction and diagnostics

These tools operate on an existing PIE/Game `UUserWidget` instance. Run the target screen, then call `ListRuntimeWidgets` with its `BlueprintPath`. Each `Instances[].RuntimeInstanceToken` identifies the current widget, world, session, and local player. Fetch a new token after PIE restarts or the widget is recreated. Legacy read-only inspection can still use `InstanceIndex`; tools requiring a runtime token cannot substitute an index. Tokens locate instances; they are not authentication.

```json
{"BlueprintPath":"/Game/UI/WBP_Inventory","Offset":0,"MaxItems":32}
```

`InspectWidgetRuntime` takes `RuntimeInstanceToken`. Give `WidgetName` to inspect one named child, or use `Offset` and `MaxItems` to page through the live WidgetTree (`MaxItems` defaults to 100 and is capped at 200). Check the target widget and window before sending input.

## Input, lists, and operation IDs

`InteractWithWidget` requires `RuntimeInstanceToken`, an exact `WidgetName`, `Action`, and a caller-generated `RequestId`. Actions are `Click`, `Wheel`, `Text`, `Drag`, `DragDrop`, and `Navigate`. Supply `WheelDelta`, `Text`, normalized `EndX`/`EndY`, `DropWidgetName`, or `Direction` and optional `Device` as the chosen action requires. Text accepts 1–128 printable BMP characters; IME, control characters, and clipboard input are not supported. Input reaches the selected live window, so confirm the page before calling it.

`ControlWidgetList` requires `RuntimeInstanceToken`, `ListName`, and `Action`. Start with `Action: "Snapshot"` to obtain an `ItemToken`. Then use `Select`, `AddSelection`, `Deselect`, `ClearSelection`, `ScrollIntoView`, `Expand`, or `Collapse`; a ScrollBox also supports `ScrollWidgetIntoView` with `TargetWidgetName`. Mutations require `RequestId`; actions other than `Snapshot` and `ClearSelection` require an `ItemToken` from the snapshot.

Use a new `RequestId` for each new action. Reuse it only for an identical retry; a reuse with conflicting parameters is rejected. For tools returning an `OperationId`, call `GetRuntimeOperation` to read `Status`, `Data`, and errors. Status may be `Pending`, `Completed`, `Failed`, `Cancelled`, `TimedOut`, or `Invalidated`; an expired or unknown ID reports `Unknown`. `CancelRuntimeOperation` accepts only a pending `OperationId`. Cancellation may still require cleanup; keep polling while `bCleanupPending` is true. Completed results have a short retention period, so do not treat an operation ID as permanent storage.

## Performance sampling and trace

`InspectWidgetPerformance` reads authored structural signals; it is not a runtime measurement. `MeasureWidgetPerformance` requires `RuntimeInstanceToken` and `RequestId`, with optional `Frames` (default 30, maximum 120) and `MaxNodes` (default 100, maximum 500). It summarizes real Slate paint, invalidation, and update events as **inclusive GameThread CPU** time per widget, not exclusive or GPU time. Query its returned `OperationId` for the result.

`CaptureWidgetTrace` requires a PIE instance with a cached Slate widget, plus `RuntimeInstanceToken` and `RequestId`. `DurationSeconds` defaults to 3 and must be 0.1–10. It starts a process-wide native Slate trace and refuses to take over another active or draining trace. Use `GetRuntimeOperation` to read `Data.TraceStatus`, `TracePath`, final bytes, and cleanup state. Filter the process-wide trace by the returned target Slate widget ID in Unreal Insights; it does not provide exclusive widget or GPU timing. A file is retained only after the writer closes and the final size is at most **16 MiB**. Oversize or cancelled output is discarded; `CleanupPending` or `CleanupUnconfirmed` is not a usable trace result.

Runtime operations do not save Widget Blueprints. Save authored changes explicitly with the relevant save tool. Treat the running editor's `tools/list` as authoritative for its available tools and schemas.
