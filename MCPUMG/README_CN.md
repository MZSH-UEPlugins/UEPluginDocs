[English](./README.md) | [中文](./README_CN.md)

# MCPUMG

MCPUMG 是一款由 MCP（Model Context Protocol，模型上下文协议）驱动的 Unreal Engine UMG AI 编辑插件。

## 概述

MCPUMG 通过运行在 Unreal Editor 内的 HTTP 服务器，向 AI 助手提供 Widget 专用的可视化编辑能力。AI 工具可以通过标准协议发现、创建和修改 Widget 树、属性、插槽、动画与事件。

## 安装与更新

1. 关闭所有正在使用目标引擎版本的 Unreal Editor 实例。
2. 选择与引擎版本匹配的 MCPUMG 安装包，将其内容复制到该引擎安装目录下的 `Engine/Plugins/Marketplace/MCPUMG`。
3. 打开复制后的 `MCPUMG.uplugin`，保留 `"Installed": true`，并添加或设置 `"EnabledByDefault": true`；打包后处理可能会移除此字段。
4. 重启编辑器。除非在项目设置中关闭 `bAutoStart`，插件会自动启动。

更新 MCPUMG 时，请先关闭编辑器，再使用匹配版本的新安装包完整替换现有 `MCPUMG` 目录。不要混用不同引擎版本的文件。

## 工具（27 个）

### 发现与读取

| 工具 | 说明 |
|------|------|
| ListWidgetBlueprints | 列出 Widget Blueprint（便捷入口） |
| GetWidgetTree | 获取 Widget 树结构 |
| GetWidgetProperties | 获取 Widget 属性 |
| SearchWidgetClasses | 搜索可用的 Widget 类 |
| CaptureWidgetScreenshot | 将 Widget 设计视图截取为 PNG |

### Widget 树编辑

| 工具 | 说明 |
|------|------|
| AddWidget | 添加 Widget |
| RemoveWidget | 删除 Widget |
| RenameWidget | 重命名 Widget 并更新相关绑定 |
| ReparentWidget | 将 Widget 移动到新的父级 |
| DuplicateWidget | 复制 Widget 子树 |
| WrapWidget | 使用新的父容器包裹 Widget |
| SetWidgetProperties | 设置 Widget 属性 |
| SetSlotProperties | 设置插槽属性（布局参数） |
| SetListViewEntryClass | 为 ListView、TileView 或 TreeView 配置经过校验的 `IUserObjectListEntry` 条目类 |

`SetListViewEntryClass` 只持久化条目类。Unreal 将 `UListView::ListItems` 标记为瞬态，因此条目数据仍应在 Blueprint 或运行时通过 `SetListItems`/`AddItem` 注入。可重复验证资产为 `/Game/MCPTests/UMG/WBP_MCPUMG_ListViewProbe` 与 `/Game/MCPTests/UMG/WBP_MCPUMG_ListEntryProbe`：对控件 `ListAssets` 调用工具，用 `GetWidgetTree` 读回 `EntryWidgetClass`，保存并重开资产，再运行现有 Construct 数据注入并截图核验。

### UMG 动画（7 个工具）

| 工具 | 说明 |
|------|------|
| ListAnimations | 列出全部动画 |
| GetAnimationDetail | 获取动画详情 |
| CreateAnimation | 创建动画 |
| DeleteAnimation | 删除动画 |
| AddAnimationTrack | 添加动画轨道 |
| RemoveAnimationTrack | 删除一个精确指定的动画属性轨道 |
| SetAnimationKeys | 设置动画关键帧 |

### 事件逻辑

| 工具 | 说明 |
|------|------|
| BindWidgetEvent | 绑定 Widget 事件 |
| UnbindWidgetEvent | 删除 Widget 事件绑定 |
| AddEventActions | 添加事件动作 |
| CreateWidgetBlueprint | 创建 Widget Blueprint |
| SetWidgetBlueprintSettings | 设置 Widget Blueprint 配置 |
| SetPropertyBinding | 设置 Widget 属性绑定 |

## 配置

项目设置 > 插件 > MCPUMG：

| 设置 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| Port | int32 | 8765 | HTTP 端口（被占用时自动递增，最多重试 10 次） |
| bAutoStart | bool | true | 编辑器启动时自动启动服务器 |
| RequestTimeoutSeconds | int32 | 30 | 工具执行超时时间 |
| CompileTimeoutSeconds | int32 | 120 | 编译超时时间 |

## MCP 连接

插件会在 Unreal Editor 内运行 HTTP 服务器，AI 客户端通过 MCP 协议连接。

**MCP URL**：`http://127.0.0.1:8765/mcp`

编辑器打开时服务器会自动启动（可通过 `bAutoStart` 配置）。如果 8765 端口被占用，插件会自动递增端口，工具栏将显示实际 URL。

### Claude Code

添加到 `~/.claude/.mcp.json`（全局）或项目根目录的 `.mcp.json`：

```json
{
  "mcpServers": {
    "ue-umg": {
      "url": "http://127.0.0.1:8765/mcp"
    }
  }
}
```

### 其他 AI 客户端

Cursor、Windsurf、VS Code Copilot 等 MCP 客户端的配置方式各不相同。服务器 URL 保持一致，具体添加 MCP 服务器的方法请查阅所用 AI 工具的文档。

## 要求

- Unreal Engine 5.2+
- 仅限编辑器
- 可选：[MCPBlueprint](https://github.com/MZSH-UEPlugins/MCPBlueprint)，用于 Blueprint 图表编辑（Widget 事件逻辑的高级图表操作会调用 MCPBlueprint 工具）

如有问题或反馈，请发送邮件至 [mengzhishanghun@outlook.com](mailto:mengzhishanghun@outlook.com)。我看到后会处理。
