[English](./README.md) | [中文](./README_CN.md)

# MCPWorldEditor

基于 MCP（Model Context Protocol）的 Unreal Engine AI 世界与关卡编辑插件。

## 概述

MCPWorldEditor 在 Unreal Editor 内运行 HTTP 服务，将 Actor 创建、变换与属性编辑、灯光、地形、关卡和视口操作暴露给 AI 助手。

> **注意**：插件仍在开发中；服务器基础设施已经可用，工具能力会持续完善。

## 配置

在 `Project Settings > Plugins > MCP World Editor` 中配置：

| 设置 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| Port | int32 | 8764 | HTTP 端口；占用时最多自动递增重试 10 次 |
| bAutoStart | bool | true | 编辑器启动时自动启动服务 |
| RequestTimeoutSeconds | int32 | 30 | 工具执行超时时间 |
| MaxRequestBodyBytes | int32 | 1048576 | UTF-8/JSON 解析前允许的最大 HTTP 请求体 |
| MaxResponseBodyBytes | int32 | 8388608 | 允许返回的最大 UTF-8 JSON 响应；截图 Base64 也受此预算约束 |
| MaxCapturePixels | int32 | 8294400 | `CaptureViewport` 允许读取的最大像素数（默认约为 4K） |

## MCP 连接

服务地址为 `http://127.0.0.1:8764/mcp`。若默认端口被占用，请以编辑器工具栏显示的实际地址为准。

### Claude Code

在全局 `~/.claude/.mcp.json` 或项目根目录 `.mcp.json` 中加入：

```json
{
  "mcpServers": {
    "ue-world-editor": {
      "url": "http://127.0.0.1:8764/mcp"
    }
  }
}
```

### 其他 AI 客户端

Cursor、Windsurf、VS Code Copilot 等客户端的配置位置不同，请按对应客户端文档添加同一 MCP 服务地址。

工具请求在进入 Game Thread 前超时会被取消，可在安全确认后重试。若请求已被 Game Thread 领取，错误内容会返回 `code=timeout_execution_in_progress`、`retryable=false` 与 `stateMayHaveChanged=true`；此时操作仍可能完成，调用方必须先查询编辑器状态，不能盲目重试。

响应大小在工具执行完成后核验。若 `tools/call` 的结果超出预算，JSON-RPC 错误数据会返回 `code=response_too_large`、`retryable=false` 与 `stateMayHaveChanged=true`；这不表示工具被回滚，调用方同样必须先核验编辑器状态。

服务器会在请求进入 Game Thread 前统一按每个工具公布的 `inputSchema` 校验顶层必填字段、精确 JSON 类型、数组元素类型、数组 `maxItems` 和未知字段；所有工具 schema 默认 `additionalProperties=false`，现有批量数组统一公布 100 项上限。业务语义（资产路径、数值范围、对象存在性等）仍由具体工具执行第二层校验。

## 要求

- Unreal Engine 5.2+
- 仅支持编辑器

## World Partition 区域加载

`LoadRegion` 通过 Unreal Engine 公开的 World Partition 加载器 API 创建并加载持久的编辑器区域。`BoundsMin` 与 `BoundsMax` 都必须包含有限的数值字段 `X`、`Y`、`Z`，且每个轴的最小值必须严格小于对应最大值。无效边界会在创建加载器之前被拒绝。

## 材质实例创建

`CreateMaterialInstance` 只接受项目 `/Game/` 内容根目录下可写的 long package name，例如 `/Game/Materials/MI_Wall`。工具会拒绝 Developers 与自动生成的 ExternalActors/ExternalObjects 目录、非法包名，以及内存或磁盘上已经存在的目标。父级可以是任何可加载的 `MaterialInterface`，包括材质实例。

## Actor 编组

`GroupActors` 会在执行变更前解析并校验完整 Actor 列表、拒绝重复目标与跨关卡编组，并直接调用 Unreal Engine 接受 Actor 数组的编组 API。工具不会为了执行操作而清空或替换编辑器 Selection，因此会保留仍然存活的 Actor 的既有选择；若一个已选中的 GroupActor 被明确取消编组并销毁，该已移除对象会自然从 Selection 中消失。

如有问题或反馈，请发送邮件至 [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)。我看到后会处理。
