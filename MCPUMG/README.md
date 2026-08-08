# MCPUMG

AI-driven UMG editing plugin for Unreal Engine, powered by MCP (Model Context Protocol).

## Overview

MCPUMG exposes Widget-specific visual editing capabilities to AI assistants via an HTTP server running inside the Unreal Editor. AI tools can discover, create, modify widget trees, properties, slots, animations, and events through a standardized protocol.

## Tools (26)

### Discovery & Reading
| Tool | Description |
|------|-------------|
| ListWidgetBlueprints | List Widget Blueprints (convenience entry point) |
| GetWidgetTree | Get widget tree structure |
| GetWidgetProperties | Get widget properties |
| SearchWidgetClasses | Search available widget classes |
| CaptureWidgetScreenshot | Capture Widget design view as PNG |

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
| SetWidgetBlueprintSettings | Set Widget Blueprint settings |
| SetPropertyBinding | Set a widget property binding |

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
