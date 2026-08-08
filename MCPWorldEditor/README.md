[English](./README.md) | [中文](./README_CN.md)

# MCPWorldEditor

AI-driven level/world editing plugin for Unreal Engine, powered by MCP (Model Context Protocol).

## Overview

MCPWorldEditor exposes level editing capabilities to AI assistants via an HTTP server running inside the Unreal Editor. AI tools can spawn actors, modify transforms, edit properties, manage lighting, landscape, and capture viewport — all through a standardized protocol.

> **Note**: This plugin is currently under development. The server infrastructure is ready, and tool capabilities continue to improve.

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

`ListActors` and `GetSelectedActors` return `ActorPath`, `FName`, `Label`, and `LevelPath`. Every tool that uses `ActorName/ActorNames` to address existing actors accepts the full `ActorPath` as the preferred exact identifier. Legacy labels and FNames execute only when they uniquely identify one actor in the current world; ambiguous input is rejected with sorted candidate paths instead of selecting, moving, editing, or deleting the wrong actor. Refresh discovery after an actor is renamed or moved to another level because its path changes.

## Requirements

- Unreal Engine 5.2+
- Editor only

## Tool and Editor State Contracts

This source revision registers 71 tools with 71 unique tool names.

`RemoveStreamingLevel` is not itself undoable. It requests that Unreal Engine preserve the existing Undo buffer, but stale references may still cause the engine to reset that buffer, so Undo-history preservation is not guaranteed. The tool removes Actor, Component, Object, and BSP selections that belong to the target level. It snapshots and verifies restoration of selected Actors, Components, and Objects outside that level; BSP selections in other levels are left untouched rather than cleared globally. If removal fails, the tool also attempts to restore the target-level selection and reports whether restoration was verified.

`MoveActorsToLevel` attempts to restore and verify the pre-call Actor, Component, Object, and BSP selection, and returns an error when complete restoration cannot be verified. It rejects the request before moving actors when Selection Lock is enabled or actor identity cannot be mapped uniquely. If the batch result is inconsistent, the tool does not issue an automatic Undo because it cannot prove that the top transaction belongs to this call; inspect both actor locations and selection state before continuing.

## World Partition Region Loading

`LoadRegion` creates and loads a persistent editor region through Unreal Engine's public World Partition loader API. `BoundsMin` and `BoundsMax` must each contain finite numeric `X`, `Y`, and `Z` fields, and every minimum coordinate must be strictly less than its matching maximum coordinate. Invalid bounds are rejected before a loader is created.

## Material Instance Creation

`CreateMaterialInstance` accepts a writable long package name under the project `/Game/` content root, such as `/Game/Materials/MI_Wall`. It rejects Developers and generated ExternalActors/ExternalObjects directories, invalid package names, and destinations that already exist in memory or on disk. The parent may be any loadable `MaterialInterface`, including a material instance.

## Actor Grouping

`GroupActors` resolves and validates the complete actor list before making changes, rejects duplicate or cross-level grouping targets, and calls Unreal Engine's actor-array grouping APIs directly. It does not clear or replace the editor selection to drive the operation, so selections of surviving actors are preserved. If an explicitly selected GroupActor is disbanded and destroyed, that removed object naturally disappears from the selection.

## Actor Attachment Transactions

`AttachActor` and `DetachActor` include both actors and the scene components that actually change hierarchy in the editor transaction, so Undo/Redo restores the component hierarchy, socket, and transform. If the engine rejects an operation or the resulting parent state does not match the request, the tool attempts to restore the original parent component, socket, and transform before returning an error. `Original attachment restored` reports whether that restoration was verified; when it is `false`, treat editor state as potentially changed and inspect it before continuing.

## Component Creation Safety

`AddComponent` accepts an exact full component class path or a short name that uniquely matches a currently loaded class; ambiguous short names are rejected with sorted full-path candidates. It rejects non-`ActorComponent`, abstract, deprecated, superseded, and `ClassWithin`-incompatible classes. Instance components follow the Unreal Editor creation order for serialization, creation notification, and registration; attachment, root assignment, or registration failure cleans up the new component and cancels the transaction.

## World Settings Editing

`GetWorldSettings` returns exact names and UE text values for every property currently editable on the instance. It no longer hides default-valued properties or presents read-only BlueprintVisible fields as writable discovery results. `SetWorldSettings` requires exact case, rejects property flags or `CanEditChange` conditions that disable editing, and pairs `PreEditChange` with `PostEditChangeProperty` inside the transaction. An import failure first restores the old value; if restoration also fails, the error reports the transaction rollback result, and a failed rollback means editor state may have changed.

## World Partition Region Lifetime

`LoadRegion` accepts only finite, strictly increasing bounds. Each axis is limited to 1,000,000 Unreal units, total volume to 1e17 cubic units, and no more than 16 MCP-created regions may remain active. A successful call returns an opaque `RegionHandle`; pass it to `UnloadRegion` to unload and release the corresponding official Editor Loader Adapter. Handles are never guessed or reused from bounds. World cleanup/map replacement and plugin module shutdown automatically release regions still owned by the plugin, after which their old handles expire.

## Restricted Console Commands

`ExecuteConsoleCommand` accepts only the seven documented `stat` commands, routes them through `GEngine->Exec`, and checks whether the engine actually handled the request; an unhandled command returns an error. Unreal `stat` commands toggle their viewport statistic display, so repeating one may turn the display off; they are not side-effect-free measurements. `Handled=true` in a successful result only means that the current engine instance accepted the command, not that the display is necessarily enabled. Some commands produce no text output.

## Material Instance Asset Persistence

`CreateMaterialInstance` accepts only a valid new package path under the project's `/Game/...` root and rejects Developers plus generated ExternalActors/ExternalObjects directories. It rejects packages already present in memory or on disk, then saves the created asset through the official `EditorAssetSubsystem`. Success requires the exact expected object path, a discoverable on-disk package, and a clean package, and returns `Saved=true`. On type, path, or persistence failure, cleanup is force-deleted only when the returned object exactly matches this call's target, and succeeds only when the target object, in-memory package, and on-disk package are all absent afterward. An unexpected object is never blindly deleted, and the error warns that partial state may remain. Asset creation itself is not a normal Undo transaction; however, failure cleanup uses Unreal Force Delete and may clear the editor's existing Undo history.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
