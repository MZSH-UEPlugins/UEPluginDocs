[English](./README.md) | [中文](./README_CN.md)

# MCPUMG

AI-driven UMG editing plugin for Unreal Engine, powered by MCP (Model Context Protocol).

## Overview

MCPUMG exposes Widget-specific visual editing capabilities to AI assistants via an HTTP server running inside the Unreal Editor. AI tools can discover, create, modify widget trees, properties, slots, animations, and events through a standardized protocol.

## Installation and Updates

1. Close every Unreal Editor instance that uses the target engine version.
2. Choose the MCPUMG package that matches the engine version, then copy its contents to `Engine/Plugins/Marketplace/MCPUMG` under that engine installation.
3. Open the copied `MCPUMG.uplugin`. Keep `"Installed": true`, and add or set `"EnabledByDefault": true`; packaged descriptors may omit this field during post-processing.
4. Restart the editor. The plugin starts automatically unless `bAutoStart` is disabled in Project Settings.

To update MCPUMG, close the editor and replace the entire existing `MCPUMG` directory with the matching new package. Do not mix files from different engine versions.

## Tools (58)

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
| InspectWidgetHitTest | Read the cached hit path at a live widget's center or local coordinates; never inject input |
| CompareUIImages | Compare two UI images without hidden resizing, cropping, or alignment |
| CaptureWidgetPreviewMatrix | Render named temporary states at named sizes as a PNG matrix |
| InspectWidgetPerformance | Read bounded authored WidgetTree complexity indicators |
| GetUIFrameworkCapabilities | Read the available CommonUI and MVVM operations |
| GetMVVMView | Read MVVM contexts and bindings on a Widget Blueprint |
| ValidateMVVMBinding | Validate one direct OneTime MVVM property relationship without editing bindings |

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
| SetWidgetLayout | Strict structured SizeBox, Canvas, Box and common Slot editing with dryRun |
| SetWidgetNavigation | Patch selected native UMG navigation directions without saving |
| GetWidgetNavigation | Read all six native UMG navigation directions |
| SetWidgetInstanceProperties | Set editable properties on a nested UserWidget instance |
| SetNamedSlotContent | Add content to one empty Named Slot |
| CaptureWidgetPreview | Render a temporary Widget instance and return an off-screen PNG |
| ExtractWidgetComponent | Copy one dependency-free widget subtree into a new Widget Blueprint |
| ApplyWidgetTheme | Preflight and apply a portable style/theme object |
| ExportWidgetTheme | Export portable styles for explicit widgets |
| BatchSetWidgetProperties | Preflight and apply bounded property changes across Widget Blueprints |
| ConfigureCommonUIWidget | Configure a supported existing CommonUI widget instance |
| AddMVVMViewModelContext | Add a supported MVVM viewmodel context |

`SetListViewEntryClass` persists only the entry class. Unreal marks `UListView::ListItems` as transient, so populate items from Blueprint or runtime data with `SetListItems`/`AddItem`.

### UMG Animations (9 tools)
| Tool | Description |
|------|-------------|
| ListAnimations | List all animations |
| GetAnimationDetail | Get animation details |
| CreateAnimation | Create animation |
| DeleteAnimation | Delete animation |
| AddAnimationTrack | Add animation track |
| RemoveAnimationTrack | Remove one precisely addressed animation property track |
| SetAnimationKeys | Set animation keyframes |
| DuplicateAnimation | Copy an animation with its tracks and widget bindings |
| SetAnimationPlaybackRange | Set an animation's playback range in seconds |

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
| ImportUIFont | Import one local TTF/OTF file as paired FontFace and UFont assets |
| SaveUIAsset | Save exactly one Texture2D, FontFace, or UFont package |

See [Reference Image to UMG](./ImageToUMG.md) for the workflow and schema. For practical navigation, component, preview, style, font, reuse, themes, animation, batch, performance, and optional framework use, see [Daily Workflow](./DailyWorkflow_EN.md), [Reuse Workflow](./ReuseWorkflow_EN.md), [Advanced Workflow](./AdvancedWorkflow_EN.md), [component extraction](./ComponentExtraction.md), [Common styles](./CommonStyles.md), [components](./Components.md), [navigation](./Navigation.md), [previews](./Preview.md), [themes](./Themes.md), [preview matrices](./PreviewMatrix.md), [animation workflow](./AnimationWorkflow.md), [batch migration](./BatchMigration.md), [performance](./Performance.md), and [optional UI frameworks](./OptionalUIFrameworks.md).

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

### Version and save troubleshooting

Layout inspection now reports parent bounds, bounded clipping ancestors and vertical text-overflow evidence, including wrapped text. `Unverified*` is not a pass. Property setters support optional `ResetToClassDefault`; structured styles add `RoundedBox` with explicit `outline`, `resource:null`, and Button `normalPadding`/`pressedPadding` (four numeric edges). Fixed corner radii require `roundingType: "FixedRadius"`; `HalfHeightRadius` produces capsules. Button style padding adds to ButtonSlot padding. See [workflow details](./ImageToUMG.md) and [layout/session guide](./LayoutAndSession.md).

`initialize` exposes `_meta.mcpumgContext` with project file/name, engine/plugin version, process, actual port and SessionId. Optionally supply `params._meta.MCPUMGExpectedSessionId` on tool calls; stale or non-string tokens fail before tool execution. Omitted tokens keep older clients compatible; this is not authentication.

`SetWidgetLayout` uses `Layout.sizeBox` (widthOverride/heightOverride, null clears), `canvas` (anchors.min/max x/y, offsets left/top/right/bottom, alignment x/y, autoSize, zOrder), `box` (size.rule Auto/Fill and positive Fill weight), and `slot` (padding, hAlign Left/Center/Right/Fill, vAlign Top/Center/Bottom/Fill). Box also accepts common padding/alignments, but duplicate fields across box/slot fail. Canvas offsets right/bottom mean sizes on unstretched axes and margins on stretched axes. It never reparents, edits parents or saves; dryRun validates only. Hit-test inspection reports cached evidence in the target window, not actual click delivery or mouse capture.

UE text structs patch existing members by default: `Padding="()"` is not zero. Use all four explicit zero edges, or `ResetToClassDefault=true` to initialize only the properties supplied in `Properties` from their Widget/Slot class CDO before importing. This does not reset other fields or inherit Blueprint instance overrides. Empty `Properties={}` is invalid. Structured style numbers must be JSON numbers; unknown nested fields are rejected before mutation. Ancestor bounds and desired text sizes are diagnostic estimates, not a substitute for viewing the final UI.

- Treat `tools/list` from the live editor as authoritative. Updating the source tree or installed package does not hot-swap the DLL in an already running editor; if the tool count differs, verify the deployment directory and restart the target editor.
- When `SaveWidgetBlueprint` fails, inspect `LogSavePackage` in the project log. Windows `Error 32` usually means a stale `-game`, PIE, or second editor process for the same project still owns the asset handle. Close only the stale process whose command line identifies that project, then retry the precise save.
- If the patch succeeded in memory but saving failed, do not use Save All, overwrite the file externally, or repeat the patch. Release the file lock and retry `SaveWidgetBlueprint` so the existing in-memory change is saved once.
- `CaptureRuntimeWidgetScreenshot` requires the MCPUMG editor module and an existing runtime Widget instance. A standalone `-game` process is not guaranteed to expose the MCPUMG server.

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

For questions or feedback, visit the [unified contact page](https://mengzhishanghun.github.io/contact/) and add me on WeChat.
