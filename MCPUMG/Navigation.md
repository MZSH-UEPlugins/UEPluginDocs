# UMG 导航工具

`SetWidgetNavigation` 为 Widget Blueprint 模板中的单个控件配置原生 `UWidgetNavigation` 数据；不会发送输入、改变运行时焦点或保存资产。调用成功后资产仅在内存中标脏，需要时再调用 `SaveWidgetBlueprint` 保存。

## 读取当前六方向

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","WidgetName":"PlayButton"}
```

调用 `GetWidgetNavigation` 会始终返回 `Up`、`Down`、`Left`、`Right`、`Next`、`Previous` 六个方向。没有配置导航对象时，六个方向均报告为 `Escape`。

## 局部更新与预检

`Rules` 只更新提供的方向，其余原有方向不变。支持 `Escape`、`Stop`、`Wrap`、`Explicit` 和 `Custom`。`Explicit` 的目标必须是同一个 `WidgetTree` 内、且不是自身的控件；`Custom` 必须提供 Widget Blueprint 生成类中已有、签名为 `UWidget* Function(EUINavigation)` 的 `CustomFunction`。

```json
{
  "BlueprintPath":"/Game/UI/WBP_Menu",
  "WidgetName":"PlayButton",
  "Rules":{
    "Down":{"Rule":"Explicit","TargetWidget":"OptionsButton"},
    "Next":{"Rule":"Wrap"}
  },
  "dryRun":true
}
```

去掉 `dryRun` 或传入 `false` 后，所有方向会先完成验证，再在一个事务中一次写入。要恢复某个方向的默认逃逸规则，只提交该方向：

```json
{"BlueprintPath":"/Game/UI/WBP_Menu","WidgetName":"PlayButton","Rules":{"Down":{"Rule":"Escape"}}}
```

无效方向、缺失/跨树/自身显式目标，或不存在的自定义函数都会在写入前失败，且不会修改任何对象。
