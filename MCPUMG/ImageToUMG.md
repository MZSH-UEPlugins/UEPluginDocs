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

## 运行版本与文件锁预检

- 每次开始前都以运行中编辑器的 `tools/list` 为准。源码仓库或安装目录已更新，不代表已经打开的编辑器进程加载了新 DLL；工具数量或能力描述不一致时，先确认部署目录，再重启目标编辑器。
- 同一项目的旧 `-game`、PIE 或第二个编辑器进程可能仍持有 `.uasset` 文件句柄。`SaveWidgetBlueprint` 失败时先查看项目日志中的 `LogSavePackage` 与系统错误码；Windows `Error 32` 表示共享冲突，应只关闭命令行明确指向同一项目的遗留进程，然后重试精确保存。
- 不用 `Save All`、外部文件覆盖或重复 Patch 绕过文件锁。Patch 已成功但保存失败时，内存态仍可能是新版本；先保留当前编辑器，再解除锁并调用 `SaveWidgetBlueprint`，不要重新提交同一写操作。
- `CaptureRuntimeWidgetScreenshot` 依赖已加载 MCPUMG 编辑器模块且已经存在目标 Widget 实例。仅以 `-game` 启动的进程不保证加载编辑器模块；需要运行态测量时，使用加载了插件的编辑器 PIE/Game 会话。

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

### 布局检查不等于布局通过

`dryRun` 只校验树和属性合法性，不执行 Slate 排版。`InspectWidgetLayout` 的 Design 模式只提供元数据；PIE 模式增加直接 Slate 父节点边界、最多 64 层裁切祖先（每页最多输出 256 条），以及包括自动换行文字在内的纵向溢出线索。`PossibleAncestorClipping` 和 `PossibleVerticalOverflow` 需要结合实际截图定位；`Unverified*` 表示没有足够证据，不是通过。

祖先矩形采用屏幕空间轴对齐包围盒，并非最终绘制裁切区。旋转、OnDemand、ClipToBoundsWithoutIntersecting、自定义绘制和缓存几何时效均可能影响判断。工具不为诊断创建实例、强制预览数据或改变运行状态。先让真实页面完成布局，再检查目标实例。

### 属性更新、清零与恢复默认

`SetWidgetProperties` 和 `SetSlotProperties` 保持默认部分更新。`Properties={}` 无效；`Padding="()"` 保留旧结构成员，不会清零。设零请显式给出四边：`(Left=0,Top=0,Right=0,Bottom=0)`。

新增可选 `ResetToClassDefault=true`：只将 `Properties` 中指定的顶层字段恢复为该 Widget/Slot 类 CDO 默认值，再导入提供的值；不影响其他字段，不修改类默认对象。它不代表零值、不读取父蓝图实例覆盖值。返回 `PropertyUpdateMode` 区分 `PatchCurrentValue` 与 `ResetToClassDefault`。

### 显式圆角和按钮内边距

`SetWidgetStyle` 的 Brush 支持 `drawType="RoundedBox"`；`outline` 是完整对象，必须提供四角、宽度、颜色及 `roundingType`。使用 `FixedRadius` 才会按四角值绘制；`HalfHeightRadius` 是胶囊模式。`resource:null` 显式清除 Brush 资源。未提供字段保留原值。

```json
{
  "Brush": {
    "drawType": "RoundedBox",
    "outline": {
      "topLeft": 4, "topRight": 4, "bottomRight": 4, "bottomLeft": 4,
      "width": 1,
      "color": {"r": 0.4, "g": 0.25, "b": 0.06, "a": 1},
      "roundingType": "FixedRadius"
    }
  }
}
```

按钮使用 `Style.Button.normalPadding` / `pressedPadding`，每个包含 `left/top/right/bottom` 四个数字，支持显式零和有限负值（有意重叠）。按钮样式 padding 与 ButtonSlot padding 相加；九宫格 Brush.margin 不是内容内边距。结构化样式的数值必须使用 JSON number，嵌套未知字段会被拒绝，整次调用在修改前校验。

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
- 持久化冲突：制造或复现第二进程文件锁，确认保存明确失败、解除锁后同一内存态可以精确保存，并且不需要重新应用 Patch。
