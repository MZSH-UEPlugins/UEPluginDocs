[English](./README.md) | [中文](./README_CN.md)

# MCP Level Sequence

MCP Level Sequence 是一个自包含的 Unreal Editor 插件，通过 MCP 兼容的 JSON-RPC 2.0 HTTP 服务，让 AI 客户端发现和编辑 Level Sequence。插件支持 Unreal Engine 5.2 及以上版本，不依赖其他用户插件或跨插件共享模块。

## 安装与连接

1. 将 `Plugins/MCPLevelSequence` 放入 UE 项目或引擎插件目录并启用插件。
2. 启动 Unreal Editor。服务默认自动监听 `http://127.0.0.1:8762/mcp`。
3. 如端口被占用，服务最多向后尝试 10 个端口；以编辑器工具栏和日志显示的实际地址为准。
4. 在 `Project Settings > Plugins > MCP Level Sequence` 中可配置端口、自动启动和请求超时。

服务支持 `initialize`、`ping`、`tools/list` 和 `tools/call`。HTTP 请求体最大为 1 MiB，超限返回 HTTP 413。请求必须使用 `jsonrpc: "2.0"`，`params` 与 `arguments` 必须是对象，并按工具 Schema 严格校验未知字段、必填字段、类型、数组元素和数值边界。

## 23 个工具

| 分组 | 工具 | 用途 |
|---|---|---|
| 序列 | `ListSequences`、`GetSequenceDetail`、`CreateSequence` | 发现、分页读取和创建序列 |
| 时间 | `SetPlaybackRange`、`SetSequenceRates` | 设置播放范围、显示帧率和 Tick Resolution |
| Binding | `ListBindings`、`AddBinding`、`RemoveBinding` | 分页列出、添加和级联删除对象绑定 |
| Track | `ListTrackTypes`、`AddTrack`、`RemoveTrack` | 查询、添加和级联删除轨道 |
| Section | `AddSection`、`RemoveSection`、`SetSectionRange` | 创建、删除和调整 Section 范围 |
| Key | `GetKeyframes`、`SetKeyframes`、`RemoveKeyframes` | 分页读取、写入和删除关键帧 |
| Camera | `AddCameraCut`、`SetCameraCut` | 创建和修改 Camera Cut |
| 生命周期 | `OpenAsset`、`SaveAsset`、`CloseAsset` | 打开、保存指定资产并核验关闭结果 |
| 预览 | `CaptureSequencerView` | 截取当前 Sequencer 视图并返回 PNG |

## Track、Section 与 Channel

`AddTrack` 支持八类轨道：

- Binding 轨道：`MovieScene3DTransformTrack`、`MovieSceneFloatTrack`、`MovieSceneBoolTrack`、`MovieSceneSkeletalAnimationTrack`。
- 根级或 Binding 轨道：`MovieSceneAudioTrack`、`MovieSceneEventTrack`。
- 仅根级轨道：`MovieSceneCameraCutTrack`、`MovieSceneFadeTrack`。

Float 与 Bool 属性轨要求 `PropertyName`，可用 `PropertyPath` 指定嵌套属性。Skeletal Animation Section 要求 `AnimationAssetPath`，并验证为 `AnimSequenceBase`；Audio Section 要求 `SoundAssetPath`，并验证为 `SoundBase`。Event Track 目前只创建 Section 容器，不创建 Director Blueprint endpoint 或事件 payload。

关键帧工具支持 Float、Double、Bool 和 Byte 四类 Channel。Transform/Fade/属性轨等实际可编辑通道以 `GetSequenceDetail` 返回结果为准。

## 时间单位与帧率

所有 `StartFrame`、排他 `EndFrame`、关键帧位置和 Camera Cut 帧都使用序列的 `TickResolution`，不是 `DisplayRate`。`GetSequenceDetail` 会同时返回这两个有理数帧率。

`SetSequenceRates` 可独立或同时设置 Display Rate 与 Tick Resolution；每个帧率必须提供正整数分子和分母。修改 Tick Resolution 时，插件调用引擎的帧时间迁移能力，同步迁移播放/选择范围、Track、Section、Key 和 Marked Frame，以保持实际时间不变；不会递归修改 Subsequence 资产。

## 分页与输出上限

- `ListSequences`：默认 100，单页最多 500。
- `ListBindings`：默认 100，单页最多 500，并按 BindingId 稳定排序。
- `GetKeyframes`：默认 200，单页最多 1000，并提供 `NextOffset`。
- `GetSequenceDetail`：用 `Offset` 与 `MaxItems` 对 Binding/Track/Section 的稳定组合遍历分页。
- `CaptureSequencerView`：最多 16 Mi 像素，压缩后的 PNG 最多 8 MiB。

分页响应会返回总数、当前 Offset、是否还有更多数据和可用的 `NextOffset`。客户端应继续分页，不要假设一次返回完整资产。

## 写入、事务与资产生命周期

修改与保存工具只接受有效的 `/Game/...` Level Sequence 路径；只读发现、读取、打开、截图和关闭可访问其他已加载或可读取的序列位置。所有编辑工具会检查 PIE、保存和 GC 忙碌状态，拒绝只读 MovieScene，并通过编辑器事务支持 Undo。

编辑成功只会将包标记为 dirty，不会隐式保存。调用 `SaveAsset` 只保存指定 Level Sequence 的 package。`CloseAsset` 发出关闭请求后会再次查询该资产及其子对象的编辑器实例；若编辑器拒绝关闭，工具会返回失败而不是误报成功。

删除 Binding 或 Track 会级联处理其下级对象，并在遇到无效、锁定或只读 Section 时拒绝破坏性操作。名称选择器存在重名时会报歧义；优先使用 `ListBindings` 返回的 `BindingId` 和 `GetSequenceDetail` 返回的 `TrackName`。

## 调用示例

下面的请求将 Display Rate 设为 24 fps，并把 Tick Resolution 迁移为 24000 fps：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "SetSequenceRates",
    "arguments": {
      "AssetPath": "/Game/Cinematics/MySequence",
      "DisplayRateNumerator": 24,
      "DisplayRateDenominator": 1,
      "TickResolutionNumerator": 24000,
      "TickResolutionDenominator": 1
    }
  }
}
```

推荐流程是先调用 `ListSequences` 和 `GetSequenceDetail`，再用稳定 ID/名称执行修改，最后显式调用 `SaveAsset`。

## 与 UE 5.8 官方实验 Toolset 的边界

UE 5.8 的实验性 Animation Assistant Toolset 是依赖 SequencerScripting 的广覆盖 Python 能力；本插件选择 UE 5.2+ 兼容、自包含的 C++ HTTP 边界。本轮已补齐可稳定跨版本实现的序列帧率设置、Skeletal Animation Track/Section 和 Audio 资产 Section。

以下差异为有意保留，而不是文档遗漏：

- Subsequence 创建/导航：需要循环依赖检查、层级所有权和跨序列时间语义设计，当前不提供写入工具。
- Spawnable 创建：对象模板所有权、Undo 与 spawner 路径跨版本变化较大，需独立运行时验证后再开放。
- Event payload：依赖 Director Blueprint endpoint/graph；当前只创建 Event Section，不承诺触发具体逻辑。
- 任意 Track/Class、Folder、Tag、选择状态和骨骼级高级操作：官方实验 Toolset 覆盖更广，本插件只公开已审查并能维持严格 Schema 的能力。

当前源码已按 UE 5.2 与 5.8 头文件和调用链进行静态核对；编译、编辑器运行、资产保存/关闭交互和完整 Undo 行为仍属于后续统一构建与运行验证边界。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
