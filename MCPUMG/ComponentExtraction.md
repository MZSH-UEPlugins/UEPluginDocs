# 提取可复用组件

`ExtractWidgetComponent` 将指定 WidgetTree 子树提取为新的 `/Game` Widget Blueprint。默认 `ReplaceSource:false` 只复制，源蓝图不变；设为 `true` 时会在一次编辑器事务中以新组件实例替换源位置，`InstanceName` 可指定实例名（默认原控件名）。目标必须是新资产路径；已有磁盘包或非空内存包都会被拒绝。

复制保留原生控件属性、普通 `bIsVariable` 标记、内部子节点层级及原生 Slot 属性。`DependencyMode` 默认 `Reject`；选择 `MigrateSupported` 可迁移子树内的 NamedSlot 内容，以及绑定全部闭合于子树、只有属性轨道且没有动态绑定函数的动画。跨子树动画、非属性轨道、图表引用、继承的 `BindWidget`/`BindWidgetOptional`、编辑器委托绑定、默认焦点、UI Components、自定义或跨子树导航等依赖仍会被拒绝。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Menu",
  "WidgetName": "CardPanel",
  "AssetPath": "/Game/UI/WBP_Card",
  "dryRun": true
}
```

先使用 `dryRun:true` 查看 `WouldChange`、`Dependencies` 及可能的 `PreflightError`；确认后改为 `false` 创建组件。新蓝图会在内存中编译，但不会自动保存。若替换源位置，还需保存源蓝图；对各自路径调用 `SaveWidgetBlueprint`。

工具不会把源页面的业务图表搬入组件。
