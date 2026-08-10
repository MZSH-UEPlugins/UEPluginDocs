[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint

MCPBlueprint is a self-contained Unreal Editor plugin that exposes Blueprint discovery, graph editing, members, components, asset lifecycle operations, and compile feedback through MCP over HTTP. It supports Unreal Engine 5.2 and later and does not depend on another user plugin.

## Connection

The default endpoint is `http://127.0.0.1:8766/mcp`. If that port is occupied, the server tries the next ports and the toolbar displays the actual endpoint. Add that URL as an HTTP MCP server in your AI client.

The server implements `initialize`, `tools/list`, `tools/call`, and `ping`. Write operations run on the game thread, participate in editor transactions where Unreal supports Undo, mark assets dirty, and do not save automatically.

Requests must use JSON-RPC `2.0` with object-valued `params` and tool `arguments`. Browser-style `Origin` headers are accepted only for `localhost`, `127.0.0.1`, or `[::1]`; non-browser clients may omit `Origin`.

## Tools (52)

### Discovery and reading

| Tool | Purpose |
|---|---|
| `ListBlueprints` | List Blueprint assets with stable bounded pagination. |
| `GetBlueprintOverview` | Read graphs, variables, functions, components, interfaces, and parent class. |
| `ListBlueprintMembers` | List functions, Custom Events, dispatchers, and local variables with unified stable pagination. |
| `GetGraphDetail` | Read graph nodes, pins, defaults, and links. |
| `SearchGraphNodes` | Search Blueprint action spawners and return stable `SpawnerId` values. |
| `GetCompileErrors` | Compile and return the authoritative Blueprint status and diagnostics. |

### User Defined Struct and Enum

| Tool | Purpose |
|---|---|
| `CreateUserDefinedStruct` / `GetUserDefinedStruct` / `ModifyUserDefinedStruct` | Create, inspect, and safely edit fields by stable GUID. |
| `CreateUserDefinedEnum` / `GetUserDefinedEnum` / `ModifyUserDefinedEnum` | Create, inspect, and safely edit enum entries. |

Use `bDryRun=true` to review the bounded reference-impact result first. Struct removal and type changes then require `bAllowPotentialDataLoss=true`. Enum removal and reordering require `bAllowSerializedValueSemanticChange=true`. Enum renaming changes only the display name and preserves the internal enumerator name. Modify operations refresh and compile referencing Blueprints, report affected DataTables/DataAssets, participate in Undo, mark assets dirty, and never save automatically. Create operations validate first; a failed partial asset is discarded and is not announced to the Asset Registry.

### Variables, functions, and events

| Tool | Purpose |
|---|---|
| `AddVariable` / `ModifyVariable` / `RemoveVariable` | Manage Blueprint member variables. |
| `CreateFunction` / `RemoveFunction` | Create or safely remove a function graph with reference checks. |
| `CreateCustomEvent` / `AddEventNode` / `AddBoundEvent` | Add custom, engine, component, or level-actor events. |
| `AddLocalVariable` / `RemoveLocalVariable` | Add or safely remove a function-local variable. |
| `AddNodePin` / `RemoveNodePin` | Add or remove supported dynamic pins on Sequence, container, and Switch nodes. |
| `AddInterface` | Implement a Blueprint interface. |
| `AddEventDispatcher` / `RemoveEventDispatcher` | Create or safely remove a multicast event dispatcher and its signature. |

### Graph editing

| Tool | Purpose |
|---|---|
| `ApplyGraphPatch` | Apply bounded node create/remove, connect/disconnect, defaults, and layout changes as one declarative transaction. |
| `SetPinDefaults` | Set validated literal defaults on existing input pins. |
| `FormatGraph` | Lay out a whole graph or a selected node set. |
| `AddCommentBox` | Enclose selected nodes in a native Comment Box with configurable title, color, and padding. |

#### Professional graph layout

`FormatGraph` uses deterministic weak-component separation, SCC cycle condensation, left-to-right layered topology, barycentric crossing reduction, measured Slate node and pin-row sizes with bounded fallbacks, pin-aware vertical alignment, and component packing. `LayoutScope` accepts `WholeGraph`, `ConnectedComponent`, or `Selection`. `LayoutStyle` accepts `Balanced` (default general-purpose spacing), `Straight` (stronger wire alignment and more vertical room), or `Compact` (smaller footprint with lighter alignment). This small preset surface is suitable for project conventions or AI skill instructions without exposing algorithm-specific weights. Whole-graph layout protects comment boxes and the nodes they currently contain. Use `bDryRun=true` to return planned positions and before/after overlap, backward-edge, crossing, long-edge, area, `FlatEdgeRatio`, `AveragePinDeltaY`, and `P95PinDeltaY` metrics without calling `Modify()` or opening an editor transaction. Optional reroute insertion is disabled by default and bounded by `MaxRerouteNodes`; its quality metrics describe the original-node plan before knot insertion.

`ApplyGraphPatch` accepts `LayoutScope=Auto|CreatedNodes|ConnectedComponent|None` and the same `LayoutStyle` presets. `Auto` formats a newly implemented function together with its structural entry/result nodes, but limits changes to newly created nodes when adding to an existing implementation. A node with an explicit `Position` remains fixed. Layout failure, reroute failure, or compile failure participates in the same patch rollback. Both tools return the applied scope, style, and layout quality metrics.

### Actor Blueprint components and defaults

| Tool | Purpose |
|---|---|
| `GetComponents` / `GetComponentProperties` | Read the local SCS hierarchy and editable component-template properties. |
| `AddComponent` / `DuplicateComponent` / `RenameComponent` / `RemoveComponent` | Manage local SCS components. |
| `ReparentComponent` / `SetRootComponent` | Edit the local SceneComponent hierarchy. |
| `SetComponentProperties` | Atomically import editable component-template values in Unreal text format. |
| `GetClassDefaults` / `SetClassDefaults` | Read paginated editable Class Default Object values, then atomically set reviewed values. |

### Asset lifecycle and capture

| Tool | Purpose |
|---|---|
| `CreateBlueprint` | Create an in-memory Blueprint asset under `/Game`. |
| `CompileBlueprint` | Compile and return authoritative status and diagnostics. |
| `SaveAsset` / `OpenAsset` / `CloseAsset` | Save or control the editor for a standalone Blueprint asset. |
| `DeleteAsset` / `RenameAsset` / `DuplicateAsset` | Manage standalone Blueprint assets. |
| `ReparentBlueprint` | Dry-run or apply a safe parent-class change with cycle and data-loss checks. |
| `CaptureGraphScreenshot` | Return a PNG and optionally save it below `Saved/MCPBlueprint/Screenshots`. |

## Safety and behavior boundaries

- New or moved Blueprint asset paths must be valid standalone package paths under `/Game`; existing in-memory and on-disk package collisions are rejected.
- Level Blueprints can be read and graph-edited by object path, but `SaveAsset`, `DeleteAsset`, `RenameAsset`, and `DuplicateAsset` do not operate on them because those actions affect the owning level package. Use a World Editor workflow for the level itself.
- `CloseAsset` reports whether an editor was open and verifies that all editors for the Blueprint and its subobjects actually closed; cancellation is returned as an error.
- Member and local variable defaults are validated with Unreal's K2 pin rules before mutation; malformed UE text values are rejected.
- Variable type changes, renames, and removals reject unsafe operations while derived Blueprint classes are unloaded. Load descendants first so inherited references can be checked; child references must be removed before deleting a declaration.
- With automatic compilation enabled, transactional Blueprint mutations return bounded compiler diagnostics and roll back when compilation does not reach `UpToDate` or `Warning`; when disabled, mutation results report `CompileStatus: Skipped`. Rollback status is reported explicitly.
- Component names are validated against the complete Blueprint member namespace. Ambiguous short component class names are rejected; use a full class path to disambiguate.
- Component classes must be Blueprint-spawnable. A new SceneComponent without an explicit parent attaches to the unique local, inherited, or native scene root; ambiguous multi-root hierarchies are rejected instead of creating a second root.
- SCS hierarchy edits snapshot the local hierarchy for Undo and roll back on compile failure. `SetRootComponent` only replaces one unambiguous local scene root, never an inherited/native root.
- `GetComponentProperties` uses `Offset`/`MaxResults` pagination (maximum 100 properties), truncates individual exported values at 8,192 characters, and applies an overall serialized property budget with continuation metadata.
- `GetClassDefaults` uses the exact editable-property filter enforced by `SetClassDefaults`, returns stable pagination with declaring-class/inheritance metadata, and bounds individual values at 8,192 characters plus an overall serialized property budget.
- Native fixed-array reflection properties (`ArrayDim > 1`) are omitted from Class Default and component-template property maps because the map protocol cannot address individual fixed-array elements safely; ordinary `TArray` properties remain supported through UE text format.
- `GetBlueprintOverview` applies stable `Offset`/`MaxResults` pagination to every section (maximum 50 requested entries per section), bounds each section by serialized output budget, bounds variable defaults at 8,192 characters, and returns at most 16 input and 16 output signature pins per function.
- `ListBlueprintMembers` uses one stable page across functions, Custom Events, dispatchers, and local variables. Signatures return at most 16 pins per semantic direction and distinguish `SignatureDirection` from the inverse physical K2 `NodeDirection` used by entry/result nodes.
- `GetComponents` returns a stable paginated flat local list and inherited list (maximum 100 entries each), plus a local hierarchy tree capped at 200 nodes and depth 64. Flat traversal uses an explicit stack and reports truncation above 10,000 components or depth 256.
- `GetGraphDetail` returns at most 25 stably ordered nodes per call under a serialized page budget. Each node exposes independently paginated visible pins (maximum 64), with a per-node pin budget, at most eight bounded links per pin, and explicit truncation/continuation metadata.
- Tool schemas reject unknown top-level arguments and validate the nested graph-patch, signature, pin-default, component-property, and class-default structures before execution.
- HTTP request bodies are rejected above 1 MiB after the engine HTTP server has received them. Serialized responses are limited to 16 MiB; tool text is limited to 1 Mi characters, and image output is limited to four images, 8 Mi base64 characters per image, and 12 Mi total.
- Compile diagnostics return at most 200 error/warning entries, 4,096 characters per message, 256 Ki characters in total, and 32 unique node GUIDs per entry. `TotalErrors`, `TotalWarnings`, returned counts, and `bTruncated` disclose omitted diagnostics. `CompileBlueprint` and `GetCompileErrors` return MCP errors for non-success terminal states; `bWarningsAsErrors` can make warnings fail the call.
- Request timeout settings bound dispatcher waiting, but synchronous Blueprint/UObject work already running on the game thread cannot be preempted. Avoid immediate retries after a client-side timeout until editor state is known.
- Blueprint listing and deletion are refused while Asset Registry discovery is running, so pagination and reference checks are not based on incomplete data.
- Destructive asset deletion is not undoable. Without `bForce`, referenced assets are refused and normal editor deletion preserves live references; forced deletion may clear references and break dependents.
- `DeleteAsset` can return `locked or in use` while the current session's Undo/Redo transaction history still retains the target. Do not mask this state with `bForce`; save work that must be kept, restart the editor, and retry with `bForce=false`.
- Screenshot file output is restricted to `Saved/MCPBlueprint/Screenshots`; absolute paths, traversal, and existing link/reparse-point paths are rejected. Existing PNG files are not replaced unless `bOverwrite=true`; these checks reduce link-path and accidental-overwrite risks but do not claim to eliminate external filesystem races.
- Source-only compatibility review is not a substitute for compiling and testing inside each target engine version.

## Configuration

Open **Project Settings > Plugins > MCP Blueprint**.

| Setting | Default | Description |
|---|---:|---|
| Port | 8766 | First HTTP port to try. |
| Auto Start | true | Start the server when the editor module loads. |
| Request Timeout | 30 seconds | Normal tool timeout. |
| Compile Timeout | 120 seconds | Compile/patch timeout. |
| Auto Compile After Modify | true | Compile after supported write operations. |

## Current Testing Focus

The current priority is complete real-editor regression testing of the existing 52 tools. This includes successful workflows, rejection paths, transaction rollback, stable pagination and output limits, compile feedback, asset persistence after save/restart, and cleanup verification. User Defined Struct/Enum tools still require real-editor and cross-version regression.

## Known limitations

- When `LayoutScope=Selection` or `ConnectedComponent` targets nodes already inside a Comment Box, the current layout planner still treats that Comment Box as an outside obstacle and may shift the selected nodes out of the box. Inspect the plan with `bDryRun=true`; the reliable workflow today is layout first, then create the Comment Box.
- An asset modified by many transactions in the same session may remain retained by Undo/Redo history and fail non-forced deletion. Follow the `DeleteAsset` safety guidance above. A future improvement will target this state without clearing unrelated undo history and will return more specific diagnostics.

## Later Roadmap

- **Member signatures and refactoring:** modify or rename function, Custom Event, and Event Dispatcher signatures and flags while preserving references safely.
- **Variable metadata:** extend variable editing to cover category, tooltip, access control, Expose on Spawn, SaveGame, replication, and RepNotify settings.
- **Graph lifecycle and impact analysis:** create, rename, and remove supported graph types; remove implemented interfaces; inspect references and affected assets before refactoring.
- **Blueprint types:** create additional Blueprint asset types such as Blueprint Interfaces, Function Libraries, and Macro Libraries.
- **Authoring and diagnostics:** improve selection layout inside Comment containers, member documentation, and debugging-oriented inspection.

These are candidate directions, not commitments to a particular release or date. They will be refined using evidence from testing and real usage.

## AI-Assisted Usage

If a tool, parameter, or workflow is unclear, you can provide this document to an AI assistant and ask it to read the documentation before attempting the operation with the MCP tools actually available in the current Unreal Editor session. Confirm target assets before write operations, and read the affected Blueprint state back afterward instead of treating a successful tool response alone as completion.

## Support

For questions, feedback, or the UE technical discussion group, use the unified contact page below.

- **Contact:** [Email, WeChat group, and X](https://mengzhishanghun.github.io/mengzhishanghun/contact/)
