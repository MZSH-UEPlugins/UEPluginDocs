# 可选 UI 框架

MCPUMG 可在项目已经启用 CommonUI 或 ModelViewViewModel（MVVM）时使用对应工具。先调用 `GetUIFrameworkCapabilities` 查看每个框架的状态：`NotInstalled`、`InstalledDisabled`、`EnabledNotLoaded` 或 `Available`。该查询不会改变项目、模块或资产状态。

若要使用可选框架，请由项目负责人按项目需要手动启用插件并重启编辑器；MCPUMG 不会代为启用插件或加载已禁用模块。

## CommonUI

`ConfigureCommonUIWidget` 只修改指定 Widget Blueprint 树里的现有实例。参数为 `BlueprintPath`、`WidgetName`、`Properties`，可选 `dryRun` 可在不修改资产前检查完整请求。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Menu",
  "WidgetName": "ConfirmButton",
  "Properties": {"bIsLocked": "True"},
  "dryRun": true
}
```

支持的控件及字段范围如下：

| 控件 | 可配置内容 |
| --- | --- |
| CommonButtonBase | Style、锁定/选择/切换行为、点击/触摸/按键方式和传统 TriggeringInputAction |
| CommonActivatableWidget | Back、自动激活、焦点/模态，以及激活/停用可见性 |
| CommonTextBlock | Style、ScrollStyle、MobileFontSizeMultiplier |

工具只接受通过控件家族白名单、编辑条件和可编辑性检查的字段；不会调用 CommonUI 函数、修改 Style 默认对象、配置 Enhanced Input、启用插件或自动保存。

## MVVM

`GetMVVMView` 读取一个 Widget Blueprint 已有的 Context 和 Binding。它只读取，不会创建或修改 MVVM 数据。

`AddMVVMViewModelContext` 为现有 Widget Blueprint 添加一个真实 ViewModel Context。`BlueprintPath`、`ViewModelClass` 和 `CreationType` 均为必填项；`CreationType` 只能是 `Manual` 或 `CreateInstance`。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "ViewModelClass": "/Game/UI/VM/BP_InventoryViewModel.BP_InventoryViewModel_C",
  "CreationType": "CreateInstance"
}
```

ViewModel 类必须实现 `NotifyFieldValueChanged`，是当前有效的具体非 Widget 类，并允许请求的创建策略。该工具不会保存资产；添加后可用 `GetMVVMView` 检查 Context，再按需要显式编译和保存。

`ValidateMVVMBinding` 是只读预检，不会增删或修改 Binding。它只验证一个直接、类型完全一致的 `OneTime` 属性关系：来源可以是 ViewModel Context 或 Widget，目标必须是现有 Widget 的公开可写运行时属性。

```json
{
  "BlueprintPath": "/Game/UI/WBP_Inventory",
  "SourceType": "ViewModel",
  "SourceName": "InventoryViewModel",
  "SourceProperty": "Title",
  "DestinationWidgetName": "TitleText",
  "DestinationProperty": "Text",
  "BindingMode": "OneTime"
}
```

函数、嵌套路径、静态数组、转换函数、隐式转换、私有或 EditorOnly 属性，以及通知驱动的绑定模式不在本验证器范围内。
