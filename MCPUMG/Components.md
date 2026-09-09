# 组件实例与 Named Slot

## 为 Named Slot 创建内容

`SetNamedSlotContent` 在指定 Widget Blueprint 的控件树中查找 `HostWidgetName`，并为其一个空的 `SlotName` 创建 `WidgetClass` 内容。宿主可以是原生实现 `INamedSlotInterface` 的控件，也可以是已经成功编译、公开 Named Slot 的嵌套 UserWidget。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Page",
  "HostWidgetName": "DetailsArea",
  "SlotName": "Header",
  "WidgetClass": "TextBlock",
  "WidgetName": "Txt_DetailsHeader",
  "bIsVariable": true,
  "dryRun": true
}
```

先以 `dryRun: true` 检查宿主、槽名、类、名称、编译状态和循环引用，再将它改为 `false` 执行。成功后使用 `GetWidgetTree` 或 `GetWidgetProperties` 读回，并显式调用 `SaveWidgetBlueprint`；工具自身不会保存。

为避免静默破坏绑定、动画或图表引用，非空槽一律拒绝。该工具也不提供 clear、replace 或 move；需要移除现有内容时，应在编辑器中核对所有依赖后处理。

## 设置嵌套 UserWidget 的实例参数

`SetWidgetInstanceProperties` 修改宿主 Widget Blueprint 控件树里某个嵌套 UserWidget 实例的可编辑字段。它不会修改被引用 Widget Blueprint 的类默认值，不会调用任意函数，也不会写入业务 Data Asset。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Page",
  "WidgetName": "CardInstance",
  "Properties": {
    "Title": "炼丹房",
    "AccentColor": "(R=0.15,G=0.8,B=0.55,A=1.0)"
  }
}
```

字段必须在该实例上可编辑，值使用 UE 文本格式。变量不必标记 `ExposeOnSpawn`；是否可写取决于编辑器属性标记和 `CanEditChange`。`ResetToClassDefault: true` 会先从嵌套控件类默认对象取得每个指定字段的默认值，再导入传入文本。空 `Properties`、结构关系字段和所有值均未变化的调用会被拒绝。

执行后读回目标实例属性，再调用 `SaveWidgetBlueprint`。保存后关闭并重新打开资产，实例参数与 Named Slot 内容应保持不变。
