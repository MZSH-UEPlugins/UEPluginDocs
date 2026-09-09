# 批量属性迁移

`BatchSetWidgetProperties` 默认 `dryRun:true`。每个目标均须提供 `ExportWidgetSpec` 返回的 `expectedFingerprint`；任一指纹过期、目标重复或属性非法时，整个请求零改动。最多 16 个 WBP、256 个控件。

实际执行按 WBP 各自一个 Undo 事务，均标为未保存；不会自动编译或保存，也不支持 Slot、关系或结构迁移。写后会逐字段反射核验；异常时结果整体为 error，并逐包报告 `Applied`、`Failed` 或 `NotAttempted`，但不会谎称跨包回滚。可对已应用包分别使用编辑器 Undo。全部目标本来已相同则报告 `NoChange`，不建立事务或弄脏包。
