# 动画工作流

`DuplicateAnimation` 深拷贝源动画的 MovieScene、轨道和 Widget 绑定，源动画不变；新名称必须唯一。`SetAnimationPlaybackRange` 以秒设置 `[StartSeconds, EndSeconds)`，所有参数先验证后在一个事务中写入。两者均只标脏，不保存。

`SetAnimationKeys` 的 Cubic 键可使用 `TangentMode`（`Auto`、`User`、`Break`）和有限数值 `ArriveTangent`、`LeaveTangent`；Linear 或 Constant 携带切线会被拒绝，且不修改资产。

`GetAnimationDetail` 同时返回 Cubic 键的切线模式与入/出切线。`Auto` 由引擎计算切线；手工斜率请使用 `User` 或 `Break`。切线必须是 JSON 数字且在 float 可表示范围内。

`GetAnimationDetail` 还分页返回绑定 GUID、轨道/区段索引及 Float、Bool、Byte 通道的键详情。先用它确定精确位置，再用 `EditAnimationKeys` 删除、清空或平移一个通道中的键。请求需要 `BindingGuid`、`WidgetName`（Slot 绑定另传 `SlotWidgetName`）、`TrackIndex`、`SectionIndex`、`ChannelType` 和 `ChannelIndex`。`Operation` 为 `Delete`、`Clear` 或 `Shift`；用整数 tick 帧数组 `Frames`，或同时给出含端点的 `StartSeconds`、`EndSeconds` 选择键。`Clear` 不带选择器时清空整个通道；`Shift` 还需 `ShiftSeconds`，目标不得碰撞现有键或超出播放范围。

`bDryRun` 默认 `true`，返回本次区段的 `Fingerprint` 和受影响键数。实际应用时传 `bDryRun:false` 与该值作为 `ExpectedFingerprint`；过期指纹会被拒绝。修改后蓝图仍需显式保存。

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","SourceAnimationName":"FadeIn","NewAnimationName":"FadeInFast"}
```

`RenameAnimation` 使用 `BlueprintPath`、现有 `AnimationName` 和唯一的 `NewAnimationName` 重命名动画；新名称不能与控件、其他动画或蓝图成员冲突。若父类原生 `BindWidgetAnim` 或 `BindWidgetAnimOptional` 属性使用旧名称，或有已知外部资产引用该 Widget Blueprint，工具会拒绝重命名；图表字符串引脚含旧动画名时也会拒绝，需要先人工复核这些依赖。重命名后仍需显式保存。

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"FadeIn","NewAnimationName":"FadeInFast"}
```

## 编辑事件关键帧

`GetAnimationDetail` 的 `RootTracks` 返回动画根轨道，包含事件轨道各区段的 `EventChannels`、`ChannelIndex`、键的 `Frame` 和 `Time`，以及可用的事件名称、端点 GUID 和编译状态。`TrackIndex` 是根轨道列表中的原始索引，不能按事件轨道过滤后重新编号。`Bindings` 中的轨道与 `RootTracks` 共用 `Offset`、`MaxItems` 和 `NextOffset`；`ContextOnly:true` 的父对象仅用于定位当前页的子项。

编辑根事件通道时，显式传入 `TrackScope:"Root"` 和 `ChannelType:"Event"`，并使用返回的 `TrackIndex`、`SectionIndex`、`ChannelIndex`。请求中不得出现 `BindingGuid`、`WidgetName` 或 `SlotWidgetName`，空字符串也不允许。省略 `TrackScope` 时仍使用 `Binding`，只支持原有的 Float、Bool、Byte 通道及绑定地址。

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","AnimationName":"Intro","TrackScope":"Root","TrackIndex":0,"SectionIndex":0,"ChannelType":"Event","ChannelIndex":0,"Operation":"Shift","StartSeconds":0.25,"EndSeconds":0.25,"ShiftSeconds":0.25,"bDryRun":true}
```

这个请求选择 0.25 秒处的事件并预览将其平移到 0.5 秒。应用时将同一请求中的 `bDryRun` 改为 `false`，并将返回的 `Fingerprint` 填入 `ExpectedFingerprint`。若轨道或区段被替换、地址重排、播放范围或区段内容发生变化，请重新预览。

`Delete` 删除所选事件；`Clear` 不带选择器时清空该事件通道。`Frames` 使用整数 tick 帧，时间范围包含两个端点。平移目标不能与未选中的键碰撞，也不能超出播放范围。编辑保留剩余事件的端点和 payload，支持 Undo/Redo；工具不会保存资产，请在确认结果后显式保存蓝图。