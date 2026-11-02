# Runtime interaction and diagnostics

These tools operate on an existing PIE/Game `UUserWidget` instance. Run the target screen, then call `ListRuntimeWidgets` with its `BlueprintPath`. Each `Instances[].RuntimeInstanceToken` identifies the current widget, world, session, and local player. Fetch a new token after PIE restarts or the widget is recreated. Legacy read-only inspection can still use `InstanceIndex`; tools requiring a runtime token cannot substitute an index. Tokens locate instances; they are not authentication.

```json
{"BlueprintPath":"/Game/UI/WBP_Inventory","Offset":0,"MaxItems":32}
```

`InspectWidgetRuntime` takes `RuntimeInstanceToken`. Give `WidgetName` to inspect one named child, or use `Offset` and `MaxItems` to page through the live WidgetTree (`MaxItems` defaults to 100 and is capped at 200). Check the target widget and window before sending input.

## Input, lists, and operation IDs

`InteractWithWidget` requires `RuntimeInstanceToken`, an exact `WidgetName`, `Action`, and a caller-generated `RequestId`. Actions are `Click`, `Wheel`, `Text`, `Drag`, `DragDrop`, and `Navigate`. Supply `WheelDelta`, `Text`, normalized `EndX`/`EndY`, `DropWidgetName`, or `Direction` and optional `Device` as the chosen action requires. Text accepts 1–128 printable BMP characters; IME, control characters, and clipboard input are not supported. Input reaches the selected live window, so confirm the page before calling it.

`ControlWidgetList` requires `RuntimeInstanceToken`, `ListName`, and `Action`. Start with `Action: "Snapshot"` to obtain an `ItemToken`. Then use `Select`, `AddSelection`, `Deselect`, `ClearSelection`, `ScrollIntoView`, `Expand`, or `Collapse`; a ScrollBox also supports `ScrollWidgetIntoView` with `TargetWidgetName`. Mutations require `RequestId`; actions other than `Snapshot` and `ClearSelection` require an `ItemToken` from the snapshot.

Default TreeView snapshots include root items and child items represented by currently generated entries. Root `Index` values keep their root-list meaning; children have `Index: -1` and `IdentitySource: "GeneratedEntry"`. Expand the parent, wait for the page to refresh, and take another Snapshot to select, scroll to, expand, or collapse generated children. A child token may become invalid when its entry scrolls out of the generated range, is recycled after collapse, or is removed. Take a new snapshot when needed; stale entry mappings cannot authorize child operations during refresh. A snapshot with `GeneratedItemsStatus: "RefreshPending"` contains only roots. Hidden children, ungenerated children, and parent relationships remain unknown; `TreeHierarchy` returns `Unknown`. Snapshot does not expand the tree or invoke child getters.

To read collapsed and ungenerated descendants, add the boolean `IncludeHierarchy: true` to a TreeView `Snapshot` request. It defaults to `false`. ListView and TileView do not support enabling it, and other actions reject the field. The request calls child getters synchronously without expanding the tree, selecting items, or generating UI entries.

```json
{"RuntimeInstanceToken":"<token>","ListName":"QuestTree","Action":"Snapshot","IncludeHierarchy":true}
```

On success, `TreeHierarchy` is `ObservedTraversal`. `Hierarchy` contains `Scope: "ProviderReturnedItems"`, `Consistency: "ObservedTraversal"`, `Complete: true`, ordered `RootIds`, and `Nodes`. Each node has `NodeId`, `ParentId`, `ChildIds`, `ChildIndex`, `Depth`, `ItemName`, and `ItemClass`. Roots have a null `ParentId`, depth 0, and a `ChildIndex` matching their root-list order. Children follow the getter's return order. A `NodeId` identifies a node only within this result and cannot serve as an `ItemToken` for selection, expansion, or other actions. `Items` still covers roots and currently generated entries.

Getters should follow the normal read-only query contract. The result describes the hierarchy observed during this traversal and does not promise an atomic view of the whole tree. Traversal allows at most 2048 unique nodes and depth 64. Null or invalid items, cycles, duplicate nodes, and multiple parents fail the request. A cooperative budget of about 100 ms is checked before and after getters and while processing nodes. A running getter cannot be interrupted, so this is not a hard request deadline. Changes to the token, widget tree, target list, root order, or cached Slate identity, and reentrant reads, terminate the request. Failed requests issue no new ItemTokens. The project implementation is responsible for honoring the read-only getter contract.

`Navigate` supports native Wrap and Custom navigation for `Next`, `Previous`, and all four arrow directions. `Device` accepts `Keyboard` or `Gamepad`. The destination must remain in the token's original widget tree and window. The effective navigation rule determines the Wrap boundary. Arrow-direction Wrap requires static rules matching the boundary determined by this call's callbacks, with no navigation simulation metadata on the source path. Other configurations are rejected.

The UMG Custom delegate and the object it is bound to must remain valid from the start of invocation until native UMG finishes converting the return value. Any non-null returned `UWidget` must also remain valid throughout that interval. Destroying these objects or allowing GC to invalidate them during this interval is unsupported. The delegate executes once. The destination must be visible, enabled, and focusable. A null return or a destination that fails the original tree, window, or focus requirements fails without falling back to automatic navigation. CustomBoundary rules and hardware/CommonUI input processing remain unsupported. Callbacks that replace the tree, rebuild Slate, change the source path or expected focus, or revoke the token fail or invalidate the operation and clear this tool's virtual focus.

Use a new `RequestId` for each new action. Reuse it only for an identical retry; a reuse with conflicting parameters is rejected. For tools returning an `OperationId`, call `GetRuntimeOperation` to read `Status`, `Data`, and errors. Status may be `Pending`, `Completed`, `Failed`, `Cancelled`, `TimedOut`, or `Invalidated`; an expired or unknown ID reports `Unknown`. `CancelRuntimeOperation` accepts only a pending `OperationId`. Cancellation may still require cleanup; keep polling while `bCleanupPending` is true. Completed results have a short retention period, so do not treat an operation ID as permanent storage.

## ComboBoxString

`ControlWidgetComboBox` supports live `UComboBoxString` widgets. Supply `RuntimeInstanceToken`, `WidgetName`, and `Action: "Snapshot"` to read options, `OptionCount`, `SelectedText`, `SelectedIndex`, and `IsOpen`. `Offset` defaults to 0; `Limit` defaults to 64 and is capped at 128. Continue with `NextOffset` and `bHasMore`. Option and selection text is capped at 4096 characters with truncation flags. Snapshot does not create Slate or open or close a popup.

```json
{"RuntimeInstanceToken":"<token>","WidgetName":"BallType","Action":"Snapshot","Offset":0,"Limit":32}
```

`Select` requires `RequestId` and exactly one of `OptionText` or `OptionIndex`. Text matching is exact and case-sensitive; duplicate matches are rejected. Indices must be non-negative integers. The popup must be closed both when the request is submitted and when it executes. An open popup produces `PopupOpen`; the tool does not close it. Selecting the same item returns `NoChange` without another broadcast. Transitions between indices with the same label, including labels that differ only in case, are unsupported. Reselecting a cleared selection can also suppress the native callback because the stored label remains unchanged; the tool rejects these transitions before calling the setter. If options change after queuing, take another Snapshot and use a new `RequestId`. Select accepts at most 2048 options, 4096 characters per option, and 65536 characters in total; larger sets are rejected.

```json
{"RuntimeInstanceToken":"<token>","WidgetName":"BallType","Action":"Select","RequestId":"choose-butterfly-1","OptionText":"蝴蝶"}
```

Before using Select, ensure callbacks meet these preconditions: synchronous `FieldNotify` callbacks must not reenter and change selection or options. Selection and content-generation callbacks must not directly or indirectly clear options, destroy or rebuild the owning instance or target widget, or release or rebuild its Slate resources. These effects are unsupported. The plugin cannot inspect arbitrary callbacks or guarantee a safe return when these preconditions are violated. Ordinary `OnSelectionChanged` may change or clear the selection within these constraints; the tool reports the actual state after the native call returns safely.

Results repeat these preconditions in `CallbackContract`. `Backend: "NativeUMG"` and `SelectionType: "Direct"` describe native semantic selection and do not prove a mouse click or Slate popup click. The tool does not inject global input or change the `ScopedSlate` boundary. Query `GetRuntimeOperation` for the final result, including state-validation errors. `bNativeCallIssued` means only that the native call was issued. If a callback changes the final state or invalidates the token, the tool does not report successful selection, replay delegates, or roll back business effects. Runtime selection has no Undo.

## Performance sampling and trace

`InspectWidgetPerformance` reads authored structural signals; it is not a runtime measurement. `MeasureWidgetPerformance` requires `RuntimeInstanceToken` and `RequestId`, with optional `Frames` (default 30, maximum 120) and `MaxNodes` (default 100, maximum 500). It summarizes real Slate paint, invalidation, and update events as **inclusive GameThread CPU** time per widget, not exclusive or GPU time. Query its returned `OperationId` for the result.

`CaptureWidgetTrace` requires a PIE instance with a cached Slate widget, plus `RuntimeInstanceToken` and `RequestId`. `DurationSeconds` defaults to 3 and must be 0.1–10. It starts a process-wide native Slate trace and refuses to take over another active or draining trace. Use `GetRuntimeOperation` to read `Data.TraceStatus`, `TracePath`, final bytes, and cleanup state. Filter the process-wide trace by the returned target Slate widget ID in Unreal Insights; it does not provide exclusive widget or GPU timing. A file is retained only after the writer closes and the final size is at most **16 MiB**. Oversize or cancelled output is discarded; `CleanupPending` or `CleanupUnconfirmed` is not a usable trace result.

Runtime operations do not save Widget Blueprints. Save authored changes explicitly with the relevant save tool. Treat the running editor's `tools/list` as authoritative for its available tools and schemas.
