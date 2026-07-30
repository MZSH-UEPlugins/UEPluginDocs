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
| MaxRequestBodyBytes | int32 | 1048576 | Maximum HTTP request body accepted before UTF-8/JSON parsing |
| MaxResponseBodyBytes | int32 | 8388608 | Maximum UTF-8 JSON response; screenshot Base64 content shares this budget |
| MaxCapturePixels | int32 | 8294400 | Maximum pixels read by `CaptureViewport` (approximately 4K by default) |

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

A tool request that times out before the Game Thread claims it is cancelled and may be retried after the editor becomes available. If the Game Thread already claimed it, the error content reports `code=timeout_execution_in_progress`, `retryable=false`, and `stateMayHaveChanged=true`; the operation may still finish, so clients must inspect editor state instead of retrying blindly.

The response-size check runs after tool execution. If a `tools/call` result exceeds the budget, JSON-RPC error data reports `code=response_too_large`, `retryable=false`, and `stateMayHaveChanged=true`; this does not mean the tool was rolled back, so clients must inspect editor state before any retry.

Before a request reaches the Game Thread, the server validates top-level required fields, exact JSON types, array item types, array `maxItems`, and unknown fields against the selected tool's published `inputSchema`; every tool schema defaults to `additionalProperties=false`, and current batch arrays publish their 100-item limit. Tool-specific business rules such as asset paths, numeric ranges, and object existence remain a second validation layer.

The editor becomes busy as soon as a PIE start request is queued. `PIEControl/GetState` and `GetEditorState` distinguish `Idle`, `StartQueued`, and `Playing`, while `Stop` can cancel a queued start. Viewport type and render-mode tools use only a known level-editor viewport instead of down-casting an asset-preview viewport; when both ViewMode and ShowFlag are supplied, both are validated before either change is applied.

## Requirements

- Unreal Engine 5.2+
- Editor only

## World Partition Region Loading

`LoadRegion` creates and loads a persistent editor region through Unreal Engine's public World Partition loader API. `BoundsMin` and `BoundsMax` must each contain finite numeric `X`, `Y`, and `Z` fields, and every minimum coordinate must be strictly less than its matching maximum coordinate. Invalid bounds are rejected before a loader is created.

## Material Instance Creation

`CreateMaterialInstance` accepts a writable long package name under the project `/Game/` content root, such as `/Game/Materials/MI_Wall`. It rejects Developers and generated ExternalActors/ExternalObjects directories, invalid package names, and destinations that already exist in memory or on disk. The parent may be any loadable `MaterialInterface`, including a material instance.

## Actor Grouping

`GroupActors` resolves and validates the complete actor list before making changes, rejects duplicate or cross-level grouping targets, and calls Unreal Engine's actor-array grouping APIs directly. It does not clear or replace the editor selection to drive the operation, so selections of surviving actors are preserved. If an explicitly selected GroupActor is disbanded and destroyed, that removed object naturally disappears from the selection.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
