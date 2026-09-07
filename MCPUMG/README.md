[English](./README.md) | [中文](./README_CN.md)

# MCPUMG

AI-driven UMG editing plugin for Unreal Engine, powered by MCP (Model Context Protocol).

## Overview

MCPUMG exposes Widget-specific visual editing capabilities to AI assistants via an HTTP server running inside the Unreal Editor. AI tools can discover, create, modify widget trees, properties, slots, animations, and events through a standardized protocol.

> Development status (2026-09-05): the reference-image-to-UMG workflow is under active development. Image understanding stays in the AI client; MCPUMG performs deterministic validation, construction, and visual verification without embedding a model or API key.

## Installation and Updates

1. Close every Unreal Editor instance that uses the target engine version.
2. Choose the MCPUMG package that matches the engine version, then copy its contents to `Engine/Plugins/Marketplace/MCPUMG` under that engine installation.
3. Open the copied `MCPUMG.uplugin`. Keep `"Installed": true`, and add or set `"EnabledByDefault": true`; packaged descriptors may omit this field during post-processing.
4. Restart the editor. The plugin starts automatically unless `bAutoStart` is disabled in Project Settings.

To update MCPUMG, close the editor and replace the entire existing `MCPUMG` directory with the matching new package. Do not mix files from different engine versions.

## Tools (37)

### Discovery & Reading
| Tool | Description |
|------|-------------|
| ListWidgetBlueprints | List Widget Blueprints (convenience entry point) |
| GetWidgetTree | Get widget tree structure |
| GetWidgetProperties | Get widget properties |
| SearchWidgetClasses | Search available widget classes |
| CaptureWidgetScreenshot | Capture Widget design view as PNG |
| CaptureRuntimeWidgetScreenshot | Capture an existing PIE/Game Widget instance at its actual Slate window geometry as PNG; never creates a widget or injects preview state |
| GetUIDesignCapabilities | Report the supported reference-to-UMG schema and safety limits |
| ExportWidgetSpec | Export a stable Widget spec, revision, and fingerprint |
| InspectWidgetLayout | Inspect authored metadata or paged live PIE layout diagnostics |
| CompareUIImages | Compare two UI images without hidden resizing, cropping, or alignment |

### Widget Tree Editing
| Tool | Description |
|------|-------------|
| AddWidget | Add widget |
| RemoveWidget | Remove widget |
| RenameWidget | Rename widget and update related bindings |
| ReparentWidget | Move widget to new parent |
| DuplicateWidget | Duplicate a widget subtree |
| WrapWidget | Wrap a widget in a new parent container |
| SetWidgetProperties | Set widget properties |
| SetSlotProperties | Set slot properties (layout parameters) |
| SetListViewEntryClass | Configure a validated `IUserObjectListEntry` class for ListView, TileView, or TreeView |
| ApplyWidgetTreePatch | Dry-run or atomically apply Widget Spec v1 operations |
| SetWidgetStyle | Apply structured Image, Border, TextBlock, or Button styles |

`SetListViewEntryClass` persists only the entry class. Unreal marks `UListView::ListItems` as transient, so populate items from Blueprint or runtime data with `SetListItems`/`AddItem`. A repeatable verification asset pair is `/Game/MCPTests/UMG/WBP_MCPUMG_ListViewProbe` and `/Game/MCPTests/UMG/WBP_MCPUMG_ListEntryProbe`: call the tool for widget `ListAssets`, read back `EntryWidgetClass` with `GetWidgetTree`, save and reopen the asset, then run the existing Construct-driven item population and capture the result.

### UMG Animations (7 tools)
| Tool | Description |
|------|-------------|
| ListAnimations | List all animations |
| GetAnimationDetail | Get animation details |
| CreateAnimation | Create animation |
| DeleteAnimation | Delete animation |
| AddAnimationTrack | Add animation track |
| RemoveAnimationTrack | Remove one precisely addressed animation property track |
| SetAnimationKeys | Set animation keyframes |

### Event Logic
| Tool | Description |
|------|-------------|
| BindWidgetEvent | Bind widget event |
| UnbindWidgetEvent | Remove a widget event binding |
| AddEventActions | Add event actions |
| CreateWidgetBlueprint | Create Widget Blueprint |
| SaveWidgetBlueprint | Compile and save exactly one Widget Blueprint without Save All or unrelated dirty packages |
| SetWidgetBlueprintSettings | Set Widget Blueprint settings |
| SetPropertyBinding | Set a widget property binding |
| ImportUITexture | Import a bounded local PNG/JPEG as a UI Texture2D |
| SaveUIAsset | Save exactly one revision-checked UI texture package |

See [Reference Image to UMG](./ImageToUMG.md) for the workflow, schema, safety boundaries, and acceptance guidance.

## Configuration

Project Settings > Plugins > MCPUMG:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| Port | int32 | 8765 | HTTP port (auto-increment if occupied, up to 10 retries) |
| bAutoStart | bool | true | Auto-start server on editor launch |
| RequestTimeoutSeconds | int32 | 30 | Tool execution timeout |
| CompileTimeoutSeconds | int32 | 120 | Compile timeout |

## MCP Connection

The plugin runs an HTTP server inside the Unreal Editor. AI clients connect via MCP protocol.

**MCP URL**: `http://127.0.0.1:8765/mcp`

The server starts automatically when the editor opens (configurable via `bAutoStart`). The toolbar shows the actual URL (port auto-increments if 8765 is occupied).

### Claude Code

Add to `~/.claude/.mcp.json` (global) or project root `.mcp.json`:

```json
{
  "mcpServers": {
    "ue-umg": {
      "url": "http://127.0.0.1:8765/mcp"
    }
  }
}
```

### Other AI Clients

MCP client configuration varies by tool (Cursor, Windsurf, VS Code Copilot, etc.). The server URL is the same — consult your AI tool's documentation for how to add an MCP server.

## Requirements

- Unreal Engine 5.2+
- Editor only
- Optional: [MCPBlueprint](https://github.com/MZSH-UEPlugins/MCPBlueprint) for Blueprint graph editing (widget event logic calls MCPBlueprint tools for advanced graph operations)

For questions or feedback, email [mengzhishanghun@outlook.com](mailto:mengzhishanghun@outlook.com). I will take care of it when I see your message.
