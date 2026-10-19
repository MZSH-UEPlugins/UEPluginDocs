# 可选 UI 框架

MCPUMG 可在项目已经启用 CommonUI 或 ModelViewViewModel（MVVM）时使用对应工具。先调用 `GetUIFrameworkCapabilities` 查看每个框架的状态：`NotInstalled`、`InstalledDisabled`、`EnabledNotLoaded` 或 `Available`。该查询不会改变项目、模块或资产状态。

若要使用可选框架，请由项目负责人按项目需要手动启用插件并重启编辑器；MCPUMG 不会代为启用插件或加载已禁用模块。

`GetUIFrameworkCapabilities.ModelViewViewModel.State` 只描述原生 MVVM；`EnhancedModule.OptInModuleDeclared`、`ModuleLoaded`、`CanWriteBindingsNow` 与 `AvailabilityReason` 才描述 `MCPUMGMVVM` 增强模块。基础版即使原生 MVVM 状态为 `Available`，也不提供完整 Binding 写入。增强版本需要包含并加载 `MCPUMGMVVM` Editor 模块，同时启用原生 `ModelViewViewModel` 插件；仅修改安装包描述文件而没有相应构建产物不能获得这些工具。以运行中编辑器的 `tools/list` 为准。

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

`ConfigureCommonUIInput` 可对现有 CommonUI Widget 的白名单输入字段做 `dryRun`，再写入既有持久资产引用。按控件类型，支持 `CommonButtonBase.TriggeringEnhancedInputAction`、`CommonActivatableWidget.InputMapping`/`InputMappingPriority`，以及 `CommonActionWidget.EnhancedInputAction`/`InputActions`；参数为 `BlueprintPath`、`WidgetName`、非空 `Properties` 和可选 `dryRun`。它只校验字段与资产引用，返回的 `InputChainStatus: NotVerifiedByAssetConfiguration` 不表示整条输入链已经可用。

运行时用 `ListRuntimeWidgets` 取得令牌后，`InspectCommonUI` 可读取实例或指定 `WidgetName`；`ControlCommonUI` 的 `Action` 可为 `Activate`、`Deactivate`、`Push`、`Remove`。`Push` 需要容器 `WidgetName` 与已编译的 `WidgetClassPath`；`Remove` 使用当前堆叠目标的 `TargetToken`。操作现有实例，不会保存 Widget Blueprint；参见[运行时交互与诊断](./RuntimeWorkflow.md)。

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

### 主动启用的 MVVM 增强模块

增强模块另提供 `GetMVVMViewDetails`、`ValidateMVVMBindingAdvanced`、`SetMVVMBinding`、`RemoveMVVMBinding`、`UpdateMVVMViewModelContext`、`RemoveMVVMViewModelContext`。先读取现有 Context、Binding 的 GUID 与 fingerprint，再用只读验证器检查完整路径。`SetMVVMBinding` 新建时省略 `BindingId`；更新时同时给出 `BindingId` 和当前 `ExpectedFingerprint`。`SourcePath`、`DestinationPath` 是按顺序列出字段名的非空数组，不能写成点分字符串。

```json
{"BlueprintPath":"/Game/UI/WBP_Inventory","SourceType":"ViewModel","SourceName":"InventoryViewModel","SourcePath":["Title"],"DestinationWidgetName":"TitleText","DestinationPath":["Text"],"BindingMode":"OneTime"}
```

增强工具支持 `OneTime`、`OneWay` 与原生 `TwoWay`。通知驱动模式需要实际 FieldNotify 描述符；`TwoWay` 两端还须可读可写、类型相同。UE 5.2–5.8 的原生编译器不接受 **TwoWay 与转换函数同时使用**，因此工具会在写入前拒绝。单向绑定可用 `SourceToDestinationConversion` 指向已有、已加载、单参数的有效转换 `UFunction`；只填该字段，不提供反向转换。写工具会编译并在失败时尝试 Undo，但不保存资产。`RemoveMVVMBinding` 必填 `BindingId` 和 `ExpectedFingerprint`；更新或删除 Context 需精确 `ViewModelId` 与期望旧值。删除被绑定引用的 Context 时，`CascadeBindingIds` 必须恰好列出全部依赖 Binding ID，否则拒绝。最后按需要显式保存该 Widget Blueprint。

UE 5.2 的原生 MVVM 无法让通知绑定跨过非末段 UObject 属性进入另一个对象的字段。增强验证器和写工具会在修改前拒绝这类 `OneWay`、`TwoWay` 路径，包括对象字段仍指向同类实例的情况；`OneTime` 仍按原有路径校验，根级 FieldNotify 字段、只经过结构体的路径及末段 UObject 字段不会因此被拒绝。UE 5.3 及以上提供相应原生机制，但具体路径和项目 MVVM 设置仍须验证。UE 5.3 中动态来源有意为 null 时仍会产生原生错误日志；对象恢复后订阅及值更新是否恢复，需单独验证。`GetUIFrameworkCapabilities.ModelViewViewModel.EnhancedModule.BindingLimits.DynamicObjectSourceNotifications` 返回当前引擎的 `EngineSupported`、`Reason` 和 `RequiresNativePathValidation`；机制受支持不表示所有路径与设置都已验证。
