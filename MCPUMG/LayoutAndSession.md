# 布局、会话与命中检查

## 连接到正确的项目

`initialize` 返回 `_meta.mcpumgContext`：ProjectFile、ProjectName、EngineVersion、PluginVersion、ProcessId、ActualPort、SessionId。端口可因占用而变化，使用返回值确认实际项目。SessionId 绑定服务器对象生命周期，重建服务器对象后改变。

后续调用可以在 **params._meta** 中携带令牌；不要把它放入业务 arguments。

```json
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"GetWidgetTree","arguments":{"BlueprintPath":"/Game/UI/WBP_Menu"},"_meta":{"MCPUMGExpectedSessionId":"从initialize读取"}}}
```

错误类型或过期令牌在工具执行前拒绝。省略令牌兼容旧客户端；这是防止误操作其他编辑器的检查，不是身份认证机制。

## 结构化布局

`SetWidgetLayout` 接受 BlueprintPath、WidgetName、Layout 与可选 dryRun。只修改指定控件或它已有的 Slot，不自动包裹、换父级、修改父控件或保存。

| Layout 区域 | 目标 | 参数 |
|---|---|---|
| sizeBox | 目标自身为 SizeBox | widthOverride、heightOverride：非负数字；null 清除该覆盖 |
| canvas | CanvasPanelSlot | anchors.min/max（x/y）、offsets 四边、alignment（x/y）、autoSize、zOrder |
| box | HorizontalBoxSlot / VerticalBoxSlot | size.rule 为 Auto/Fill，Fill 可带正 weight；padding、hAlign、vAlign |
| slot | Button/Border/Overlay/SizeBox/ScrollBox/HorizontalBox/VerticalBox Slot | padding、hAlign、vAlign |

hAlign 为 Left/Center/Right/Fill，vAlign 为 Top/Center/Bottom/Fill。padding 和 offsets 都使用 left/top/right/bottom 四个 JSON 数字。不能在 box 和 slot 重复指定相同字段。

```json
{
  "BlueprintPath":"/Game/UI/WBP_Menu",
  "WidgetName":"StartButton",
  "dryRun":true,
  "Layout":{"canvas":{
    "anchors":{"min":{"x":0,"y":0},"max":{"x":0,"y":0}},
    "offsets":{"left":20,"top":30,"right":170,"bottom":60},
    "alignment":{"x":0,"y":0},"autoSize":false,"zOrder":2
  }}
}
```

Canvas 未拉伸的轴：right/bottom 是宽高；拉伸轴：right/bottom 是原生 UMG 的右/下边距。坐标相对于直接父容器，不是整个屏幕。sizeBox null 清除覆盖并不等于宽高0；省略字段保留当前值。

dryRun 验证参数、类型和适用目标，不执行实际排版；成功不证明无重叠。正式执行把 dryRun 改为 false，读回 GetWidgetProperties，验证后 SaveWidgetBlueprint 精确保存。

## 解释裁切与缩放

使用 InspectWidgetLayout 的 PIE 模式读取实际尺寸、布局缩放、父节点边界、裁切祖先和文本溢出线索。Design 模式不能给出真实排版结论。Unverified 不表示通过；旋转包围盒、条件裁切、缓存过期等仍需结合截图。

Button 样式 NormalPadding 与 ButtonSlot Padding 相加。子控件不能直接使用外层 SizeBox 的尺寸作为可用空间；应核对内部实际布局。

## 检查按钮为什么无法命中

`InspectWidgetHitTest` 接受 BlueprintPath、WidgetName 和可选 InstanceIndex。默认检查目标中心；可成对提供 LocalX/LocalY（目标自身局部坐标）。只查询目标窗口的缓存命中网格，不移动鼠标、不发送点击、不改变焦点。

结果包含 bTargetOnHitPath、目标启用/可见性、键盘焦点及最多64个 Slate 路径节点。TargetNotOnCachedHitPath 可能来自遮挡、禁用、命中可见性或缓存状态，不能直接等同于某一种故障。其他窗口、鼠标捕获和实际事件执行不在该诊断范围内。没有运行实例或有效几何时明确失败。

建议顺序：确认会话 → 读取布局 → 检查命中 → 调整布局/属性 → 读回 → 真实交互测试 → 精确保存。

## 验证

在插件测试工程执行 Automation 的 `MCPUMG` 组。包括实际 Slate 排版裁切/换行高度、遮挡/禁用命中网格、结构化布局、属性重置、样式解析和会话令牌检查。测试资产 `/Game/MCPTests/UMG/WBP_MCPUMG_LayoutContracts` 用于保存重开回归。
