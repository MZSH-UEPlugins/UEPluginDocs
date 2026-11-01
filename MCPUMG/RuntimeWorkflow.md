# 运行时交互与诊断

这些工具操作已经存在的 PIE/Game `UUserWidget` 实例。先运行目标页面，再用 `ListRuntimeWidgets` 的 `BlueprintPath` 查找实例；返回的 `Instances[].RuntimeInstanceToken` 绑定当前 Widget、World、会话和本地玩家。PIE 重启或 Widget 重建后重新取令牌。`InstanceIndex` 仍可用于旧版只读检查，但需要运行时令牌的工具不可用索引代替。令牌用于定位，不是身份认证。

```json
{"BlueprintPath":"/Game/UI/WBP_Inventory","Offset":0,"MaxItems":32}
```

`InspectWidgetRuntime` 使用 `RuntimeInstanceToken` 读取命名子控件（`WidgetName`），或用 `Offset`、`MaxItems` 分页读取运行中 WidgetTree；`MaxItems` 默认 100，最大 200。实际输入操作前，先检查目标控件和窗口状态。

## 输入、列表与操作编号

`InteractWithWidget` 必填 `RuntimeInstanceToken`、精确 `WidgetName`、`Action` 和调用者生成的 `RequestId`。`Action` 为 `Click`、`Wheel`、`Text`、`Drag`、`DragDrop` 或 `Navigate`；分别按需要提供 `WheelDelta`、`Text`、归一化终点 `EndX`/`EndY`、`DropWidgetName`，或 `Direction` 与可选 `Device`。文本限制为 1–128 个可打印 BMP 字符；此工具不提供 IME、控制字符或剪贴板输入。输入作用于目标运行时窗口，执行前请确认它正是要操作的页面。

`ControlWidgetList` 必填 `RuntimeInstanceToken`、`ListName` 和 `Action`。先用 `Action: "Snapshot"` 取得列表项 `ItemToken`；随后可 `Select`、`AddSelection`、`Deselect`、`ClearSelection`、`ScrollIntoView`、`Expand`、`Collapse`，或对 ScrollBox 使用 `ScrollWidgetIntoView` 与 `TargetWidgetName`。除 `Snapshot` 外，修改操作需要 `RequestId`；除 `Snapshot`、`ClearSelection` 外，需要快照中的 `ItemToken`。

每个新动作使用新的 `RequestId`；仅在重试完全相同的请求时复用。参数冲突的复用会被拒绝。启动后返回 `OperationId` 的工具可用 `GetRuntimeOperation` 查询 `Status`、`Data` 和错误；状态可能是 `Pending`、`Completed`、`Failed`、`Cancelled`、`TimedOut`、`Invalidated`，过期或未知编号返回 `Unknown`。`CancelRuntimeOperation` 只接受仍在等待的 `OperationId`。取消可能仍需清理，`bCleanupPending` 表示尚未确认最终状态；继续查询直至结束。已完成结果只短暂保留，不应把 `OperationId` 当永久记录。

## ComboBoxString

`ControlWidgetComboBox` 只支持运行中的 `UComboBoxString`。传入 `RuntimeInstanceToken`、`WidgetName` 和 `Action: "Snapshot"`，读取选项、`OptionCount`、`SelectedText`、`SelectedIndex` 和 `IsOpen`。`Offset` 默认 0，`Limit` 默认 64，最大 128；用 `NextOffset` 和 `bHasMore` 翻页。选项与当前选择的文本最多返回 4096 个字符，截断时带对应标记。Snapshot 不创建 Slate，也不打开或关闭弹窗。

```json
{"RuntimeInstanceToken":"<token>","WidgetName":"BallType","Action":"Snapshot","Offset":0,"Limit":32}
```

`Select` 必须提供 `RequestId`，并且只提供 `OptionText` 或 `OptionIndex` 中的一个。文本按大小写精确匹配，重复文本会被拒绝；索引须为非负整数。Select 要求弹窗在请求时和执行时都处于关闭状态，打开时返回 `PopupOpen`，不会替调用者关闭。选择同项返回 `NoChange`，不额外广播；同标签不同索引的转换，包括仅大小写不同的标签，均不受支持。清空当前选择后重选原标签，也可能被原生字符串判等跳过委托；工具会在调用前拒绝这类转换。选项在排队后发生变化时，请重新读取 Snapshot 并使用新的 `RequestId`。Select 最多处理 2048 项，每项最多 4096 个字符，合计最多 65536 个字符，超限会拒绝。

```json
{"RuntimeInstanceToken":"<token>","WidgetName":"BallType","Action":"Select","RequestId":"choose-butterfly-1","OptionText":"蝴蝶"}
```

调用 Select 前须确认回调满足以下前提：同步 `FieldNotify` 回调不重入修改当前选择或选项；选择和内容生成回调不直接或间接清空选项、销毁或重建所属实例或目标控件，也不释放或重建其 Slate 资源。这些场景不受支持，插件无法预检任意回调，也不能保证违反前提的原生调用安全返回。普通 `OnSelectionChanged` 在这些前提内可改选或清空当前选择，工具按安全返回后的实际状态报告结果。

结果中的 `CallbackContract` 会重复说明这些前提。`Backend: "NativeUMG"`、`SelectionType: "Direct"` 表示原生语义选择，不能作为鼠标点击或 Slate 弹窗点击成功的证据。工具不会注入全局输入，也不会改变 `ScopedSlate` 的作用域。用 `GetRuntimeOperation` 查询最终结果；状态预检错误也会记录在操作中。`bNativeCallIssued` 只表示原生调用已发出；回调改变最终状态或使令牌失效时不报告选择成功，不重试委托，也不回滚业务副作用。运行时选择没有 Undo。

## 性能采样和 Trace

`InspectWidgetPerformance` 读取设计态结构复杂度，不能代替运行时测量。`MeasureWidgetPerformance` 必填 `RuntimeInstanceToken`、`RequestId`；可设 `Frames`（默认 30，最大 120）和 `MaxNodes`（默认 100，最大 500）。它汇总实际 Slate paint、invalidation、update 事件的逐 Widget **包含式 GameThread CPU** 时间，不是独占时间或 GPU 时间。用返回的 `OperationId` 查询结果。

`CaptureWidgetTrace` 仅用于有缓存 Slate Widget 的 PIE 实例，必填 `RuntimeInstanceToken`、`RequestId`，`DurationSeconds` 可选（默认 3，范围 0.1–10 秒）。它启动进程级原生 Slate Trace；已有 Trace 正在运行或收尾时不会接管。结束后用 `GetRuntimeOperation` 查看 `Data.TraceStatus`、`TracePath`、实际字节数及清理状态。Trace 包含整个进程，需在 Unreal Insights 中按返回的目标 Slate Widget ID 过滤；它不是该 Widget 的独占或 GPU 计时。文件最终仅在已关闭且不超过 **16 MiB** 时保留，超限或取消会丢弃；`CleanupPending` / `CleanupUnconfirmed` 表示还不能把文件当成可用结果。

运行时操作不会保存 Widget Blueprint。设计态修改仍需显式调用相应保存工具；能力与参数以当前编辑器的 `tools/list` 为准。
