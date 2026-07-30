[English](./README.md) | [中文](./README_CN.md)

# MCPWorldEditor

AI-driven level/world editing plugin for Unreal Engine, powered by MCP (Model Context Protocol).

## Overview

MCPWorldEditor exposes level editing capabilities to AI assistants via an HTTP server running inside the Unreal Editor. AI tools can spawn actors, modify transforms, edit properties, manage lighting, landscape, and capture viewport — all through a standardized protocol.

> **Note**: This plugin is currently under development. The server infrastructure is ready, but tools are being implemented.

## Configuration

Project Settings > Plugins > MCP World Editor:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| Port | int32 | 8764 | HTTP port (auto-increment if occupied, up to 10 retries) |
| bAutoStart | bool | true | Auto-start server on editor launch |
| RequestTimeoutSeconds | int32 | 30 | Tool execution timeout |

## MCP Connection

The plugin runs an HTTP server inside the Unreal Editor. AI clients connect via MCP protocol.

**MCP URL**: `http://127.0.0.1:8764/mcp`

The server starts automatically when the editor opens (configurable via `bAutoStart`). The toolbar shows the actual URL (port auto-increments if 8764 is occupied).

### Claude Code

Add to `~/.claude/.mcp.json` (global) or project root `.mcp.json`:

```json
{
  "mcpServers": {
    "ue-world-editor": {
      "url": "http://127.0.0.1:8764/mcp"
    }
  }
}
```

### Other AI Clients

MCP client configuration varies by tool (Cursor, Windsurf, VS Code Copilot, etc.). The server URL is the same — consult your AI tool's documentation for how to add an MCP server.

## Requirements

- Unreal Engine 5.2+
- Editor only

## World Partition Region Loading

`LoadRegion` creates and loads a persistent editor region through Unreal Engine's public World Partition loader API. `BoundsMin` and `BoundsMax` must each contain finite numeric `X`, `Y`, and `Z` fields, and every minimum coordinate must be strictly less than its matching maximum coordinate. Invalid bounds are rejected before a loader is created.

## Material Instance Creation

`CreateMaterialInstance` accepts a writable long package name under the project `/Game/` content root, such as `/Game/Materials/MI_Wall`. It rejects Developers and generated ExternalActors/ExternalObjects directories, invalid package names, and destinations that already exist in memory or on disk. The parent may be any loadable `MaterialInterface`, including a material instance.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
