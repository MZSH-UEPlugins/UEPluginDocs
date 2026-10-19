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
