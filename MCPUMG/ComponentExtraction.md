# 提取可复用组件

`ExtractWidgetComponent` 将指定 WidgetTree 子树复制为新的 `/Game` Widget Blueprint，不删除、替换、修改或保存源蓝图。目标必须是新资产路径；已有磁盘包或非空内存包都会被拒绝。

复制保留原生控件属性、普通 `bIsVariable` 标记、内部子节点层级及原生 Slot 属性。以下不能独立迁移的依赖会被拒绝：图表引用、继承的 `BindWidget`/`BindWidgetOptional`、编辑器委托绑定、动画绑定、默认焦点、UI Components、NamedSlot 内容、自定义导航函数，以及指向子树外的控件或显式导航引用。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Menu",
  "WidgetName": "CardPanel",
  "AssetPath": "/Game/UI/WBP_Card",
  "dryRun": true
}
```

先使用 `dryRun:true` 检查源控件、依赖与目标路径；确认后改为 `false` 创建组件。新蓝图会在内存中编译，但不会自动保存。需要持久化时对新路径调用 `SaveWidgetBlueprint`。

工具不会把源页面的业务图表搬入组件，也不会自动用新组件替换源子树。
