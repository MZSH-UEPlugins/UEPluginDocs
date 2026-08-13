[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint

MCPBlueprint is a self-contained Unreal Editor plugin that exposes Blueprint discovery, graph editing, members, components, asset lifecycle operations, and compile feedback through MCP over HTTP. It supports Unreal Engine 5.2 and later and does not depend on another user plugin. Win64 editor packages with precompiled binaries are verified across UE 5.2–5.8.

For Fab-ready product copy, the current compatibility matrix, installation, seven end-to-end workflows, privacy, troubleshooting, and verified media, see [FAB_LISTING.md](./FAB_LISTING.md).

## Connection

The default endpoint is `http://127.0.0.1:8766/mcp`. If that port is occupied, the server tries the next ports and the toolbar displays the actual endpoint. Add that URL as an HTTP MCP server in your AI client.

The server implements `initialize`, `tools/list`, `tools/call`, and `ping`. Write operations run on the game thread, participate in editor transactions where Unreal supports Undo, mark assets dirty, and do not save automatically.

Requests must use JSON-RPC `2.0` with object-valued `params` and tool `arguments`. Browser-style `Origin` headers are accepted only for `localhost`, `127.0.0.1`, or `[::1]`; non-browser clients may omit `Origin`.

## Tools (53)

### Discovery and reading

| Tool | Purpose |
|---|---|
| `ListBlueprints` | List Blueprint assets with stable bounded pagination. |
| `GetBlueprintOverview` | Read graphs, variables, functions, components, interfaces, and parent class. |
| `ListBlueprintMembers` | List functions, Custom Events, dispatchers, and local variables with unified stable pagination. |
| `GetGraphDetail` | Read graph nodes, pins, defaults, and links. |
| `SearchGraphNodes` | Search Blueprint action spawners by Unreal's English standard names and return stable `SpawnerId` values. Standard queries include `Branch`, `For Loop`, `For Each Loop`, `Switch on Int/Name/String`, and `Add/Subtract/Multiply/Divide`. |
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
| `AddVariable` / `ModifyVariable` / `RemoveVariable` | Manage Blueprint member variables. `ModifyVariable` also updates `Category` and `Tooltip` transactionally. |
| `CreateFunction` / `RenameFunction` / `RemoveFunction` | Create, safety-gated rename, or safely remove a function graph. Rename defaults to dry-run impact analysis and requires explicit approval before reference updates. |
| `CreateCustomEvent` / `AddEventNode` / `AddBoundEvent` | Add custom, engine, component, or level-actor events. |
| `AddLocalVariable` / `RemoveLocalVariable` | Add or safely remove a function-local variable. |
| `AddNodePin` / `RemoveNodePin` | Add or remove supported dynamic pins on Sequence, container, and Switch nodes. |
| `AddInterface` | Implement a Blueprint interface. |
| `AddEventDispatcher` / `RemoveEventDispatcher` | Create or safely remove a multicast event dispatcher and its signature. |

For `RenameFunction`, omit `bDryRun` (or set it to `true`) to inspect the stable graph GUID, bounded local/external reference counts, affected Blueprints, unloaded derived classes, and blockers without mutation. Apply only by sending both `bDryRun=false` and `bApproveReferenceUpdates=true`. The first version deliberately rejects override/protected and RepNotify functions, incomplete reference scans, unloaded derived Blueprints, `CreateDelegate` bindings, and AnimBlueprint/AnimGraph state found in the declaring Blueprint, loaded dependents, or Asset Registry referencers—even when ordinary node-reference counting is zero. This gate is fail-closed because UE 5.4+ can rewrite opaque nested function property bindings that cannot yet be scanned or restored transactionally. A successful apply preserves `FunctionGraphGuid`, verifies the old name has no remaining references, compiles every affected Blueprint, participates in Undo, marks packages dirty, and never saves automatically.

### Graph editing

| Tool | Purpose |
|---|---|
| `ApplyGraphPatch` | Apply bounded node create/remove, connect/disconnect, defaults, and layout changes as one declarative transaction. |
| `SetPinDefaults` | Set validated literal defaults on existing input pins. |
| `FormatGraph` | Lay out a whole graph or a selected node set. |
| `AddCommentBox` | Enclose selected nodes in a native Comment Box with configurable title, color, and padding. |

#### Professional graph layout

`FormatGraph` uses deterministic weak-component separation, SCC cycle condensation, left-to-right layered topology, barycentric crossing reduction, measured Slate node and pin-row sizes with bounded fallbacks, pin-aware vertical alignment, and component packing. `LayoutScope` accepts `WholeGraph`, `ConnectedComponent`, or `Selection`. `LayoutStyle` accepts `Balanced` (default general-purpose spacing), `Straight` (stronger wire alignment and more vertical room), or `Compact` (smaller footprint with lighter alignment). This small preset surface is suitable for project conventions or AI skill instructions without exposing algorithm-specific weights. Whole-graph layout protects comment boxes and the nodes they currently contain. Use `bDryRun=true` to return planned positions and before/after overlap, backward-edge, crossing, long-edge, area, `FlatEdgeRatio`, `AveragePinDeltaY`, and `P95PinDeltaY` metrics without calling `Modify()` or opening an editor transaction. Optional reroute insertion is disabled by default and bounded by `MaxRerouteNodes`; its quality metrics describe the original-node plan before knot insertion.

`ApplyGraphPatch` accepts `LayoutScope=Auto|CreatedNodes|ConnectedComponent|None` and the same `LayoutStyle` presets. MCP-driven placement is required to avoid overlap by default and must not silently accept an unresolved collision. A node with an explicit `Position` remains fixed. Layout failure, reroute failure, or compile failure participates in the same patch rollback. Both tools return the applied scope, style, and layout quality metrics.

`Patch.MoveNodes` provides a bounded, explicit path for moving existing nodes when a new business block needs space or a complex graph needs staged reorganization. Each entry is `{ "NodeGuid": "...", "Position": { "X": 1200, "Y": 600 } }`. Coordinates are absolute graph units. The operation preserves each node GUID, class, pins, defaults, and every existing link; all requested moves share the graph-patch transaction, compile gate, rollback, and single Undo. Read GUIDs and coordinates with `GetGraphDetail` first. Unknown or duplicate GUIDs, non-finite/out-of-range positions, and requests beyond the shared 60-operation patch limit are rejected without partial changes.

When `Auto` falls back to `CreatedNodes` inside an existing implementation, it first anchors each created connected block between its real external predecessor and successor. If that block needs more horizontal space, only a bounded colliding right-side closure is shifted rigidly along `+X`; every node in the closure receives the same delta and keeps its `Y`, while nodes outside the closure keep their GUIDs, coordinates, and connections. The patch fails closed rather than moving a function entry, crossing Comment Box containment, exceeding the node/iteration/distance limits, or introducing any new backward edge. Explicit `LayoutScope=CreatedNodes` intentionally retains the older below-existing placement and does not move existing nodes. Write tools still do not auto-save: read back and compile first, then call `SaveAsset` explicitly. One editor Undo restores both created nodes and the displaced closure.

### Layout-pressure evidence and readable-logic acceptance

The following UE 5.2 captures remain available as **layout-pressure evidence only**. They prove bounded `+X` displacement, one-step Undo, and save/close/reopen persistence. They do **not** represent a real business function or pass the readable-logic acceptance bar: their 102/108-node setup must not be used as an example of normal production wiring. The replacement acceptance run uses the connected 112-node `BP_ConnectedWorkflow100.EvaluateBatchWorkflow`, a batch scoring and risk-filtering function with For Each, Branch, For Loop, Switch, and connected arithmetic/result chains. A registry-backed Real/Double Add and Clamp were inserted into the normalization path, producing `Multiply -> Add calibration -> Clamp [0,1] -> Set NormalizedScore`. Only the seven-node right-side closure moved to make room; the other 103 business nodes retained their coordinates and connections. Read-back reported 114 nodes with zero isolated nodes, compile completed with zero errors/warnings, save/close/reopen preserved the graph, one editor Undo restored the exact 112-node baseline, and Redo restored the 114-node result. Fixed-frame before/after/Undo captures plus readable local and complete final-stage views are archived as AIHub task outputs; `Saved/` remains intentionally outside source control.

![102-node layout-pressure baseline](./Images/LayoutShift/01_Baseline_102Nodes.png)

*Layout-pressure baseline only; not a real connected business-logic example.*

![Three arithmetic nodes inserted with bounded local displacement](./Images/LayoutShift/03_Insert_ArithmeticChain_AutoShift.png)

*Pressure-test stage 2 (`103 → 106`): nine old nodes moved by the same `+633` graph units with `DeltaY=0`. This proves local displacement only, not business-logic readability.*

![Persisted arithmetic insertion pressure-test view](./Images/LayoutShift/03_ArithmeticChain_Local.png)

*Saved/reopened pressure-test view; it is retained as local-displacement evidence, not a readable workflow example.*

![A second local insertion moves a smaller closure](./Images/LayoutShift/04_Insert_AddPair_AutoShift.png)

*Pressure-test stage 3 (`106 → 108`): two old nodes moved by the same local `+298`; compile, Undo, save, close, and reopen passed. It is not a business-logic acceptance image.*

![Persisted two-Add pressure-test view](./Images/LayoutShift/04_AddPair_Local.png)

*Saved/reopened pressure-test view only; the [stage-1 local view](./Images/LayoutShift/02_Insert_Add_Local.png) remains evidence that a single Add fit without moving old nodes.*

For a readable real workflow with 100+ nodes, require connected Exec and data paths. Branch, For Loop, For Each Loop, Switch, and Add/Subtract/Multiply/Divide results must feed a later assignment, condition, or return—not merely exist as separate nodes. Create a legal member-variable, function-parameter, or local-variable Get near each consuming region; do not fan out a Function Entry pin or distant getter across the entire graph. A temporary calculation is not safely replaceable with a duplicated Get: keep its semantics with a local-variable Set/Get pair or a bounded reroute. `ApplyGraphPatch` does not silently rewrite old logic or old connections outside the requested patch; `Auto` may only change the coordinates of the explicitly reported bounded right-side closure described above. Use dry-run/read-back to review placement and wiring before applying a mutation.

For standard arithmetic, call `SearchGraphNodes` with the English Unreal action name `Add`, `Subtract`, `Multiply`, or `Divide`, then pass the exact returned `Operator:<Name>` value to `ApplyGraphPatch`; never construct a SpawnerId manually. In that same patch, connect a Real/Double source pin to `A` or `B` and set the node's optional `PromotedType` to `double` or `real`. Unreal's schema performs its native connection-driven promotion; after all connections, the tool verifies that the operator function and `A`, `B`, and `ReturnValue` are Real/Double, otherwise the entire patch is rolled back. Unreal's UI and serialized pin text can use **Float**, **Double**, or **Real** wording depending on engine version and context; this workflow explicitly guarantees the Real/Double form, not an independently forced single-precision Float form.

For standard flow control, search with the exact English Unreal names `Branch`, `For Loop`, `For Each Loop`, `Switch on Int`, `Switch on Name`, or `Switch on String`. Loop results are real StandardMacros Action Registry entries (for example `Macro:/Engine/EditorBlueprintResources/StandardMacros.StandardMacros:ForLoop`), while Branch and the three scalar switches use their registry-backed `Native:K2Node_*` IDs. Always use the returned value; fabricated macro or native IDs remain invalid. The editor UI can localize node titles after placement even though search uses stable English names.

![Standard flow-control nodes created from SearchGraphNodes results](./Images/Fab/FlowControl_StandardNodes.png)

*Real UE 5.2 capture after Search → ApplyGraphPatch → read-back → compile → save → close/reopen. The six standard nodes are separated with no overlap.*

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
- `ModifyVariable` accepts optional `Category` and `Tooltip` for member variables. An empty `Category` restores Unreal's default category; an empty `Tooltip` removes its metadata. The tool updates the variable declaration transactionally, marks the Blueprint modified without forcing a structural rebuild, participates in Undo, rolls back on compile failure, and never saves automatically. `GetBlueprintOverview` returns `Tooltip` when present.
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

The current UE 5.2–5.8 Win64 BuildPlugin matrix passed. Refreshed packages were deployed to UE 5.2–5.6 and UE 5.8 with matching DLL hashes, version-specific BuildIds, and `EnabledByDefault=true`. The refreshed UE 5.7 package is built but its redeployment is pending because an unrelated editor currently locks the installed DLL; the previously deployed version-matched build had already passed EnginePlugin loading and MCP health. MCP initialize, `tools/list` (53), and ping passed on UE 5.2, 5.3, and 5.5–5.8. UE 5.4 reached EnginePlugin load, 53-tool registration, and endpoint startup.

## Verified media

The images in [`Images/Fab`](./Images/Fab/) are uncomposited PNGs captured from real Unreal Blueprint graph widgets. Captions and workflow context are maintained in [FAB_LISTING.md](./FAB_LISTING.md). They contain no local filesystem paths and are not presented as HTTP-response or variable-Details screenshots. The arithmetic pair shows the persisted Real/Double `Add → Subtract → Multiply → Divide → Result` chain in two truthful views: the zoom-to-fit capture covers the left/middle chain, while a node-focused capture covers `Subtract → Multiply → Divide → Result` because UE 5.2's off-screen graph-widget capture can omit far-right node bodies during a full-graph frame.

The current 53-tool source has passed UE 5.2 compilation and the automated `RenameFunction` safety regression, including persisted Asset Registry referencers, GUID conflicts, `CreateDelegate`, AnimBlueprint/AnimGraph, override, RepNotify, explicit approval, Undo recovery, and injected-failure rollback. Real-editor HTTP verification covered initialize, `tools/list` (53), ping, default dry-run, approval-gated rename with a real external caller, stable Graph GUID retention, read-back, Blueprint compilation, save, close/reopen persistence, and screenshots. The six User Defined Struct/Enum tools have also passed real-editor HTTP regression for create/read/modify workflows, stable GUID retention, dry runs, explicit approval gates, rejection paths, pagination, and direct Map-value cycle detection. The current Win64 BuildPlugin matrix succeeded for UE 5.2–5.8; refreshed deployment and DLL-hash verification passed on UE 5.2–5.6 and UE 5.8, with UE 5.7 pending release of an unrelated editor's DLL lock.

Packaging verification proves that every target engine can compile the plugin and receive the expected deployed layout; it is not a substitute for exercising every MCP operation in every editor version. Broader cross-version real-editor regression of the remaining tool surface, save/restart persistence, and cleanup behavior remains ongoing.

## Known limitations

- `Auto` local displacement is deliberately bounded to 128 existing nodes, 128 closure iterations, and 100000 graph units. It fails closed at Function Entry or Comment containment boundaries instead of expanding into an unsafe whole-graph rewrite; use a smaller insertion or explicitly reorganize the protected region.
- When `LayoutScope=Selection` or `ConnectedComponent` targets nodes already inside a Comment Box, the current layout planner still treats that Comment Box as an outside obstacle and may shift the selected nodes out of the box. Inspect the plan with `bDryRun=true`; the reliable workflow today is layout first, then create the Comment Box.
- An asset modified by many transactions in the same session may remain retained by Undo/Redo history and fail non-forced deletion. Follow the `DeleteAsset` safety guidance above. A future improvement will target this state without clearing unrelated undo history and will return more specific diagnostics.

## Later Roadmap

Development has resumed in bounded, independently verified units. The current descriptive-variable-metadata unit covers `Category` and `Tooltip`; the entries below are remaining directions, not commitments to a specific version or date.

- **Member signatures and refactoring:** safely modify function signatures and flags, then extend the same model to Custom Event and Event Dispatcher signatures.
- **Remaining variable metadata:** extend variable editing to cover access control, Expose on Spawn, SaveGame, replication, and RepNotify settings.
- **Graph lifecycle and impact analysis:** create, rename, and remove supported graph types; remove implemented interfaces; inspect references and affected assets before refactoring.
- **Blueprint types:** create additional Blueprint asset types such as Blueprint Interfaces, Function Libraries, and Macro Libraries.
- **Authoring and diagnostics:** improve selection layout inside Comment containers, member documentation, and debugging-oriented inspection.

These are candidate directions, not commitments to a particular release or date. They will be refined using evidence from testing and real usage.

## AI-Assisted Usage

If a tool, parameter, or workflow is unclear, you can provide this document to an AI assistant and ask it to read the documentation before attempting the operation with the MCP tools actually available in the current Unreal Editor session. Confirm target assets before write operations, and read the affected Blueprint state back afterward instead of treating a successful tool response alone as completion.

## Support

For questions, feedback, or the UE technical discussion group, use the unified contact page below.

- **Contact:** [Email, WeChat group, and X](https://mengzhishanghun.github.io/mengzhishanghun/contact/)
