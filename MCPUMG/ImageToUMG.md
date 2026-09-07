# 参考图到 UMG 工作流

MCPUMG 不在插件内部连接图片模型，也不保存模型密钥。AI 客户端负责理解参考图、核对文字、拆分可复用组件并生成结构化规格；MCPUMG 只负责确定性地校验、构建、测量和保存 Unreal Engine 资产。

## 工作流

1. 调用 `GetUIDesignCapabilities` 读取当前 Schema、限制和支持的 Widget/样式能力。
2. AI 根据参考图生成 `schemaVersion=1` 的 Widget 规格，并明确设计尺寸、布局假设和素材来源。
3. 调用 `ImportUITexture` 导入 PNG/JPEG；默认禁止覆盖已有资产。
4. 调用 `ApplyWidgetTreePatch` 的 `dryRun` 预演完整树或增量操作。
5. `dryRun` 返回模拟结果；正式执行继续使用预演前的 `expectedRevision`、`expectedFingerprint`，把 `dryRun` 改为 `false` 后重新获取 canonical `inputHash`，再执行同一组 operations。
6. 使用 `SetWidgetStyle` 应用结构化 Brush、文字或按钮状态样式。
7. 使用 `ExportWidgetSpec` 和 `GetWidgetTree` 读回结构与属性。
8. 使用 `InspectWidgetLayout` 检查屏幕像素边界、裁剪、文本溢出和资源状态。
9. 使用 `CaptureWidgetScreenshot` 或 `CaptureRuntimeWidgetScreenshot` 取得设计态或运行态画面。
10. 使用 `CompareUIImages` 与参考图比较，AI 根据差异生成下一轮 Patch。
11. 验证完成后分别调用 `SaveWidgetBlueprint` 和 `SaveUIAsset` 精确保存目标包。

## Widget Spec v1

顶层规格包含：

- `schemaVersion`：固定为 `1`。
- `BlueprintPath`：唯一目标 Widget Blueprint。
- `expectedRevision`、`expectedFingerprint`：乐观并发门禁。
- `requestId`、`inputHash`：幂等调用标识。
- `dryRun`：预演或正式执行；改变该字段后必须重新计算 `inputHash`。
- `operations`：`create`、`set`、`move`、`rename`、`remove`。

节点以稳定名称标识，包含 `class`、`parent`、`order`、`properties` 和 `slot`。插件在修改前验证单根、循环、容器兼容性、名称、属性、Slot、节点数量、深度和累计文本预算。错误应返回精确 JSON Pointer。

## 样式与素材

`SetWidgetStyle` 使用 JSON 对象，不要求调用方拼接 UE 文本结构：

- Brush：资源、绘制方式、图像尺寸、颜色、九宫格 Margin、平铺和 UV。
- Text：颜色和字号。
- Button：Normal、Hovered、Pressed、Disabled 四种 Brush。

纹理导入显式设置 sRGB、UI 压缩、无 Mip 和 UI Texture Group。材质只能引用已经存在且实际 `MaterialDomain == MD_UI` 的 Material Instance；插件不会根据图片任意生成材质图。

## 安全边界

- `dryRun` 不修改真实 Widget Blueprint，不编译、不保存。
- 单次 Patch 只操作一个 Widget Blueprint；跨包操作不宣称原子性。
- 修改前完整演算最终树，成功后只刷新和编译一次。
- 删除或重命名会同步同一 Widget 树内可安全迁移的导航绑定；遇到外部蓝图图表、动画、事件、变量、C++ `BindWidget` 等无法证明安全的依赖时拒绝执行。
- 图片文件、解码像素、Widget 数量、层级、请求文本和响应正文均有硬预算。
- 任何保存操作只接受一个精确目标包，不执行 Save All。
- 参考图不能唯一表达交互、遮挡内容和响应式规则；AI 必须在规格中记录假设，插件不猜测业务逻辑。

## 验收建议

- 非法输入：验证零变化、无编译、无保存。
- 并发：旧 revision/fingerprint 必须被拒绝。
- 幂等：相同 `requestId + inputHash` 重试不得重复修改。
- 回滚：故障注入后设计树和 GeneratedClass 必须恢复一致。
- 持久化：保存、关闭并重开后再次导出规格核对。
- 视觉：至少验证 1280×720、1920×1080、2560×1440，以及不同宽高比。
- 图片：同尺寸、同 sRGB 和明确透明合成背景；禁止自动裁剪、缩放或对齐掩盖布局偏差。
